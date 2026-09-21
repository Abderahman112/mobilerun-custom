# AI Phone Agent — Comparative Report
## mobilerun (Qwen3.6-27B) vs mobile_use (GUI-Owl-32B)

**Date:** July 14–15, 2026
**Prepared by:** Abdelrahman Awagih
**Compared against:** Mobile-Agent-v3.5 (`mobile_use`)
**Task suite:** Identical 3-task benchmark + extended social media testing

---

## 1. System Overviews

### Our System — mobilerun + Qwen3.6-27B

mobilerun is an open-source AI agent framework that controls an Android device via ADB and an optional Portal companion app. It reads the phone's UI using the Android accessibility tree (element indices, text labels, coordinates) and captures screenshots every step when vision is enabled.

**Core loop (current architecture):**
```
Accessibility tree + screenshot → send both to LLM → LLM emits XML tool call
→ execute via ADB → repeat until complete()
```

**Model:** Qwen3.6-27B-NVFP4, self-hosted on vLLM at `<vLLM-server>:8000/v1`
**Primary input:** Both accessibility tree and screenshot — model picks whichever is more informative
**Vision:** Always on — screenshot sent every step alongside the tree

### Their System — mobile_use + GUI-Owl-32B

`mobile_use` is a vision-language-model driven agent that operates via screenshot-first perception. Every step it captures a screenshot, extracts on-screen text via uiautomator, feeds both to the VLM, and executes the resulting action.

**Core loop:**
```
Screenshot + UI text extract → send to VLM → parse action
→ validate → execute via ADB → record → repeat
```

**Model:** GUI-Owl-1.5-32B-Instruct, NVFP4 quant, self-hosted on vLLM
**Primary input:** Screenshot + extracted UI text (vision-first)
**Memory:** Persistent cross-screen text buffer in prompt

---

## 2. Architecture Comparison

| Feature | mobilerun (Ours) | mobile_use (Theirs) |
|---------|-----------------|---------------------|
| Primary perception | Accessibility tree + screenshot (both every step) | Screenshot + UI text |
| Vision | Always on — screenshot sent every step | Every step |
| Perception fallback | If tree is blocked/sparse, model uses screenshot coordinates | OCR fallback for canvas/game text |
| Cross-app memory | Agent `<add_memory>` tags | Persistent capped text buffer |
| Completion verification | Model calls `complete()` | Screen-evidence check before accepting success |
| Stuck detection | ✅ 3x same action OR alternating A/B/A/B over 6 steps → redirect + fresh screenshot | ✅ 3x same action → auto-Back |
| Parallel tool calls | ❌ Disabled (sequential, one action at a time) | Not specified |
| Human-like typing | ✅ Char-by-char, 20–50ms delays | ✅ Char-by-char via ADB Keyboard |
| Tap realism | Standard ADB tap + show_touches enabled | Gaussian jitter + bounds clamp |
| Scroll realism | Standard | Randomized column, height, length per scroll |
| Step budget awareness | ✅ Remaining steps injected each turn | ❌ Not mentioned |
| Safety gate | ❌ No dry-run or confirmation | ❌ No dry-run or confirmation |
| Checkpoint/resume | ❌ | ❌ |
| Parallelism | Single device, linear | Single device, linear |
| Non-ASCII input | Portal keyboard (reliable) | ADB Keyboard |
| Max steps | 40 | Not specified |

---

## 3. Head-to-Head Benchmark — Task Results

All 3 tasks were identical. Same device, same apps, same day.

### Task 1 — YouTube Login Check

| Metric | mobilerun + Qwen3.6-27B | mobile_use + GUI-Owl-32B |
|--------|------------------------|--------------------------|
| Result | ✅ Success | ✅ Success |
| Time | **2m 32s (152s)** | **30s** |
| Steps | 3 | ~1–2 |
| Answer accuracy | ✅ Correct account name | ✅ Correct |
| Speed winner | | ✅ **5x faster** |

### Task 2 — Like a Specific YouTube Video

| Metric | mobilerun + Qwen3.6-27B | mobile_use + GUI-Owl-32B |
|--------|------------------------|--------------------------|
| Result | ✅ Success | ✅ Success |
| Time | **2m 37s (157s)** | **41s** |
| Steps | 2 (after prompt fix) | ~2–3 |
| Issue encountered | Looped on like button (accessibility text didn't update) | None noted |
| Speed winner | | ✅ **~4x faster** |

### Task 3 — Video Status Report via Email

| Metric | mobilerun + Qwen3.6-27B | mobile_use + GUI-Owl-32B |
|--------|------------------------|--------------------------|
| Result | ✅ Full success | ⚠️ Partial success |
| Time | **8m 57s (537s)** | **8m 31s (512s)** |
| Steps | 12 | Similar |
| Arabic title in email | ✅ Reproduced verbatim | ❌ Rendered in English |
| Data accuracy | ✅ All 5 fields correct | ⚠️ Fidelity failure on non-Latin text |
| Quality winner | ✅ **Better output quality** | |
| Speed winner | Roughly equal | |

### Overall Benchmark

| Metric | mobilerun + Qwen3.6-27B | mobile_use + GUI-Owl-32B |
|--------|------------------------|--------------------------|
| Tasks fully completed | **3 / 3 (100%)** | **2.5 / 3 (~83%)** |
| Task 1 time | 152s | **30s** |
| Task 2 time | 157s | **41s** |
| Task 3 time | 537s | 512s |
| Total time | ~846s (~14 min) | **~583s (~10 min)** |
| Avg time per step | ~49s | **~15–30s** |
| Arabic verbatim accuracy | ✅ Pass | ❌ Fail |
| False success reported | 0 | 0 (harness-enforced) |

---

## 4. Extended Testing — Post-Benchmark Runs

After the initial benchmark, additional runs were conducted to stress-test the system with raw (unscripted) prompts and new app categories. These runs exposed real failure modes and drove the architecture improvements documented in Section 6.

### Task 3 Re-runs (Raw Prompt, No Pre-setup)

The benchmark Task 3 used a pre-structured prompt ("You are already in the YouTube app…"). The following runs used a raw prompt with no setup, which is harder and more realistic.

#### Run A — Failed (22 steps, parallel tools bug)

**Prompt:** `Go to this video https://youtu.be/DteHnBZy0a4 and report its status back to me using an Email. <recipient email>`

| Phase | Steps | What happened |
|-------|-------|--------------|
| Chrome opened correctly | 1–2 | Chose Chrome over Samsung Internet ✅ |
| Video data collected | 3–4 | Title, channel, views, likes captured ✅ |
| Gmail compose — parallel tools failure | 9 | Model sent 5 actions simultaneously: type email → click index 4 (hit "Attach files" instead of Subject) → type subject into wrong field → click Send → type body. All 5 ran blind with no result checking. |
| Trapped in photo picker | 10–22 | Spent 13 steps trying to dismiss the attachment dialog that parallel tools accidentally opened |
| vLLM server crash | 23 | `APIError: EngineCore encountered an issue` — server went offline |

**Root cause:** `parallel_tools: true` let the model fire 5 actions without seeing intermediate results. One wrong index click cascaded into a 13-step trap.
**Fix applied:** `parallel_tools: false` — model now executes one action, reads result, then decides next.

#### Run B — Failed (22 steps, Samsung Internet + vision-only mode bug)

**Prompt:** Same as Run A

| Phase | Steps | What happened |
|-------|-------|--------------|
| Samsung Internet opened | 2 | App opener chose `com.sec.android.app.sbrowser` — no explicit browser instruction in prompt |
| Address bar loop | 3–6 | Model tapped same address bar coordinates 3 times (vision-only, no tree, couldn't confirm focus) |
| URL typed, video loaded | 7–10 | Loop detector fired, model switched to index click, found URL bar, navigated correctly |
| Gmail compose — subject field loop | 18–20 | With tree suppressed (vision-only mode), model guessed y=360 for Subject field (actual: y=783). Same wrong coordinate 3 times. |
| Timeout | 22 | 1000s task timeout hit |

**Root cause:** At this point the accessibility tree was being suppressed whenever vision was on. This broke Gmail compose — the tree has exact labels for Subject/To/Body fields, and without it the model was guessing coordinates from the screenshot and getting them wrong.
**Fix applied:** Accessibility tree restored alongside screenshots. Both sent every step. Model uses tree indices for standard apps (Gmail, Chrome) and screenshot coordinates when the tree is sparse (blocked apps).

#### Run C — Partial success (23 steps, outbox trap)

**Prompt:** Same as Run A

| Phase | Steps | What happened |
|-------|-------|--------------|
| YouTube opened directly | 1 | Model chose YouTube app, not a browser ✅ |
| URL pasted into search bar | 2–3 | Model clicked Search, pasted URL — YouTube treated it as a search, didn't navigate |
| Recovered to Chrome | 4–5 | Pivoted to Chrome, dismissed notification, loaded video ✅ |
| Data collected | 6 | Title (Arabic), channel, 60K likes, 1.1M views, 2.3K comments, 3 days ago ✅ |
| Gmail compose — clean | 7–14 | Sequential, index-based. Compose → recipient → confirm → subject → body → Send. No loops. ✅ |
| Outbox trap | 15–19 | Gmail showed "Sending…" then "4 unsent in Outbox / No connection". Model waited 11s then clicked Retry. |
| Retry bug | 20–23 | Gmail's Retry opened a blank new compose window instead of retrying the queued email. Model started re-composing from scratch — creating a duplicate. Run was manually aborted. |

**Root cause:** Device had no internet connection at send time. Gmail queued the email (it will send when connection restores). The model didn't know queued = success, and Gmail's Retry UI is misleading.
**Fix applied:** System prompt rule — "Outbox / Sending… / No connection after tapping Send = success. Call `complete(success=true)` immediately. Never click Retry."

---

### Task 4 — Like First 5 Posts on X (Twitter)

**Prompt:** `Open the first 5 X posts and like them and report back what the posts are about`
**Result:** ✅ 4/5 posts liked — ⚠️ 5th post timed out

| Step range | What happened |
|-----------|--------------|
| 1 | Opened X app (com.twitter.android) ✅ |
| 2–3 | Scrolled to dismiss language overlay, oriented to timeline |
| 4–6 | Post 1 opened and liked: Borg — $1,000 football prediction contest (England vs Argentina) ✅ |
| 7–11 | Post 2 opened and liked: Gothamist — Buckled Midtown tower, no peer review ✅ |
| 12–14 | Post 3 opened and liked: New York Knicks — One month championship anniversary ✅ |
| 15–18 | Post 4 opened and liked: The New Yorker — "Sort the items" puzzle ✅ |
| 19–23 | Post 5 (Jim Farley — Romain Dumas wins Goodwood Hillclimb) partially visible at bottom of screen. Model tried to scroll to reveal the Like button in the timeline instead of tapping to open the post. Same swipe 3 times → loop detector fired → 1000s timeout hit before recovery. |

**Performance:**
- Steps to like posts 1–4: 14 steps (3.5 steps/post — very efficient)
- Strategy used: tap post to open → click Like → click Back → repeat (index-based, reliable)
- Strategy abandoned on post 5: tried to like from timeline without opening, scroll loop ensued

**Fix applied:** Loop detector redirect now includes: "If scrolling isn't revealing what you need, TAP the item to open it instead of scrolling further."

**Key observation:** X (Twitter) is a React Native app. The accessibility tree is sparse (generic View/TouchableHighlight elements, no meaningful labels for feed content). The model correctly identified post text from the screenshot and used tree indices only for well-labelled elements (Like button, Back button). This confirms the dual tree+screenshot architecture works correctly for social media apps.

---

## 5. Detailed Dimension Comparison

| # | Dimension | mobilerun + Qwen3.6-27B | mobile_use + GUI-Owl-32B |
|---|-----------|------------------------|--------------------------|
| 1 | **Task success rate** | 100% benchmark (3/3); 4/5 on social media run | ~83% (2.5/3) |
| 2 | **Data-extraction accuracy (English)** | High | High |
| 3 | **Multilingual accuracy (Arabic)** | ✅ Reproduced verbatim | ❌ Translated to English |
| 4 | **Hallucination rate** | Low (memory via add_memory tags) | Reduced by cross-screen buffer; not eliminated |
| 5 | **App choice accuracy** | ✅ Correct under explicit instructions; defaults to Samsung Internet without guidance | ✅ Correct under explicit; failed on abstract "electronic message" |
| 6 | **Element-targeting accuracy** | Good on standard elements; coordinate guessing fails when tree suppressed | Good on standard; risky on small targets (pickers/sliders) |
| 7 | **Step efficiency** | Moderate (avg 5.7 steps/benchmark task; 3.5 steps/social post) | Better (fewer steps on simple tasks) |
| 8 | **Speed — simple tasks (T1, T2)** | ~150s each | **~30–41s each** |
| 9 | **Speed — complex tasks (T3)** | 537s | 512s (roughly equal) |
| 10 | **Avg inference per step** | ~49s | **~15–30s** |
| 11 | **Cross-app reliability** | ✅ YouTube → Gmail succeeded; X timeline reliable | ⚠️ App-choice and hallucination errors concentrated here |
| 12 | **Instruction robustness** | Needs explicit phrasing; raw prompts expose navigation gaps | Needs explicit phrasing; abstract instructions caused failures |
| 13 | **Recovery from stuck state** | ✅ Loop detector (3x same + A/B/A/B alternating) → redirect + screenshot | ✅ Loop detector → auto-Back |
| 14 | **Completion honesty** | Model self-reports (can false-positive) | Harness screen-evidence check (more reliable) |
| 15 | **Human realism — taps** | Standard ADB + show_touches enabled (visible on screen) | Gaussian jitter + bounds clamp |
| 16 | **Human realism — scrolls** | Standard | Randomized per scroll |
| 17 | **Human realism — typing** | ✅ Char-by-char, 20–50ms random delays | ✅ Char-by-char, ADB Keyboard |
| 18 | **Safety / confirmation gate** | ❌ None | ❌ None |
| 19 | **Privacy** | ✅ Fully local | ✅ Fully local |
| 20 | **Cost per task** | $0 (local GPU) | $0 (local GPU) |
| 21 | **Open-source / modifiable** | ✅ Yes | ✅ Yes |
| 22 | **Model swappable** | ✅ Any OpenAI-compatible endpoint | ✅ Any OpenAI-compatible endpoint |
| 23 | **Non-ASCII text input** | ✅ Portal keyboard (reliable) | ✅ ADB Keyboard |
| 24 | **Social media apps** | ✅ Tested on X — 4/5 posts liked, tree+screenshot hybrid works | Not tested in benchmark |
| 25 | **Checkpoint / resume** | ❌ | ❌ |

---

## 6. Architecture Evolution

The system went through three perception architecture iterations during testing. Each change was driven by a real failure observed in a run.

### v1 — Accessibility Tree Only (baseline)
**Perception:** Tree only. Vision off by default, escalates after 3 stuck steps.
**Problem found:** Samsung Internet opened for YouTube links, played video fullscreen, completely blocked the accessibility tree (3 useless elements). Model alternated click/swipe for 40 steps — never got vision because the alternating pattern bypassed the 3-consecutive detector.

### v2 — Vision Always On, Tree Suppressed
**Perception:** Screenshot every step. Tree suppressed entirely when vision active.
**Problem found:** Broke Gmail compose. Without the tree, the model guessed coordinates for the Subject field and got them wrong repeatedly (y=360 vs actual y=783). Standard Google apps have excellent accessibility trees — suppressing them was counterproductive.

### v3 — Both Always On (current)
**Perception:** Tree + screenshot every step. Model uses whichever is more informative.
- Rich tree (Gmail, Chrome, YouTube app) → model clicks by index (fast, reliable)
- Sparse/blocked tree (Samsung Internet, games, banking apps, React Native social media) → model reads screenshot coordinates
- No per-app configuration needed — a blank tree is self-evidently useless, a full tree speaks for itself

**Additional fixes applied alongside v3:**
- `parallel_tools: false` — prevents blind multi-action cascades
- System prompt: Chrome preferred over Samsung Internet; YouTube app preferred over browser for YouTube links
- System prompt: Gmail Outbox after Send = success, never retry
- Loop detector redirect: if scroll loop detected, tap to open item instead
- Alternating A/B/A/B loop detection added (6-step sliding window)

---

## 7. Where Each System Wins

### mobilerun + Qwen3.6-27B wins on:
- **Multilingual accuracy** — Arabic text reproduced verbatim; GUI-Owl translated it to English despite a fidelity rule against it.
- **Overall task accuracy** — 100% benchmark vs ~83%; 80% on social media (4/5 posts, 5th timed out rather than wrong).
- **Social media compatibility** — tested on X (Twitter); tree+screenshot hybrid handles React Native apps where the tree is sparse.
- **Architectural flexibility** — modifiable Python codebase; all improvements above were made by editing source files directly.
- **Step budget awareness** — remaining steps injected every turn keeps the model goal-focused near the limit.

### mobile_use + GUI-Owl-32B wins on:
- **Speed on simple tasks** — Task 1: 30s vs 152s (5x). Task 2: 41s vs 157s (4x). Significant advantage for short tasks.
- **Completion verification** — screen-evidence check before accepting success is more reliable than model self-report.
- **Human realism** — tap jitter and randomized scroll geometry make interactions harder to detect as automated.
- **Honest failure handling** — harness-enforced; our system relies on model self-reporting.

---

## 8. Root Cause Analysis

### Why GUI-Owl is faster on simple tasks
GUI-Owl-32B is purpose-trained as a GUI agent. It sees a screenshot and immediately knows what to tap. Qwen3.6-27B is a general-purpose reasoning model — it reasons about the task rather than pattern-matching to GUI actions, which takes more tokens per step. GUI-Owl's per-step latency (~15–30s) is roughly 2x lower than Qwen (~49s).

### Why Qwen handles Arabic better
GUI-Owl's fidelity rule exists at the prompt level but the model ignored it for Arabic. Qwen3.6-27B has stronger multilingual text handling from training. This is a model capability difference, not a code difference.

### Why Task 3 times are similar
Task 3 involves real-world waiting (app switching, typing, form filling). The inference speed advantage of GUI-Owl is small relative to total task time, so both land at ~8.5–9 minutes.

### Why parallel tools caused a cascade failure
With `parallel_tools: true`, the model can emit multiple actions in one response. Without seeing intermediate results, it assumed element indices remain stable across actions. Index 4 happened to be "Attach files" not "Subject" — one wrong click opened the photo picker and trapped the agent for 13 steps. Sequential execution (one action → observe result → next action) prevents this class of failure entirely.

### Why Samsung Internet keeps being chosen
The device's default browser is Samsung Internet. When the user says "go to a URL" without specifying Chrome, the app opener picks the system default. Samsung Internet intercepts YouTube links and plays them fullscreen inside the browser, blocking the accessibility tree. The fix is a system prompt rule that explicitly names Chrome as the required browser.

---

## 9. Shared Weaknesses (Neither System Solved)

- **No safety gate** — both systems will send real emails, post likes, and make irreversible changes without asking. Critical missing feature for production use.
- **No checkpoint/resume** — a crash or timeout loses all progress.
- **Single device, linear** — no parallelism across multiple devices or tasks.
- **Abstract instructions fail** — both need explicit, unambiguous phrasing. "Send an electronic message" fails; "Send a Gmail to X" succeeds.
- **Small UI targets** — sliders, time pickers, and small icons are unreliable in both.
- **Network-dependent actions** — neither handles device offline state gracefully (e.g. Gmail Outbox).
- **App default selection** — both systems can choose wrong apps when instructions are ambiguous.

---

## 10. Recommended Improvements

In priority order:

1. **Add completion verification** — before accepting `complete(success=true)`, capture a fresh screenshot and confirm the screen matches the expected end state. Eliminates false successes.

2. **Add tap jitter** — Gaussian noise (±5–15px) on every tap coordinate. Makes interactions harder to detect as automated.

3. **Add randomized scroll geometry** — randomize swipe start column and swipe distance each call.

4. **Add a confirmation gate** — before any irreversible action (send email, post, like, purchase), pause and ask. Neither system has this; it's the biggest missing safety feature for production.

5. **Handle network-dependent failures explicitly** — detect "No connection" states and either wait/retry intelligently or report them clearly rather than looping.

6. **Consider a GUI-specialist model** — if speed is the priority, swap Qwen3.6-27B for a purpose-trained GUI VLM (GUI-Owl, SeeClick). Speed on simple tasks would drop from ~150s to ~30–40s.

---

## 11. Which Model to Choose

| Use Case | Recommendation |
|----------|---------------|
| Speed is critical (simple tasks under 60s) | **mobile_use + GUI-Owl-32B** |
| Multilingual content (Arabic, non-Latin) | **mobilerun + Qwen3.6-27B** |
| Maximum task accuracy | **mobilerun + Qwen3.6-27B** |
| Complex multi-app workflows | **mobilerun + Qwen3.6-27B** (better cross-app success) |
| Social media automation | **mobilerun + Qwen3.6-27B** (tested; tree+screenshot hybrid handles React Native) |
| Human-realism / anti-detection | **mobile_use + GUI-Owl-32B** (tap jitter, randomized scrolls) |
| Honest failure reporting | **mobile_use + GUI-Owl-32B** (harness-enforced) |

**Bottom line:** GUI-Owl-32B is the better choice if speed on short tasks is the priority and content is English. Qwen3.6-27B is the better choice if accuracy, multilingual support, complex task completion, or social media automation matters more. Neither is production-ready without a safety confirmation gate.

---

## 12. Full Performance Data

### Benchmark (scripted prompts)

| Task | Steps | Time | Outcome |
|------|-------|------|---------|
| Task 1 — YouTube login check | 3 | 2m 32s | ✅ Logged in (account verified) |
| Task 2 — Like specific video | 2 | 2m 37s | ✅ Liked (55,571 likes) |
| Task 3 — Email video status report | 12 | 8m 57s | ✅ Full report sent, Arabic verbatim |
| **Benchmark total** | **17** | **~14m** | **3/3 (100%)** |

### Extended runs (raw prompts)

| Run | Task | Steps | Outcome | Failure mode |
|-----|------|-------|---------|-------------|
| A | Task 3 raw prompt | 22 | ❌ Failed | Parallel tools → photo picker trap → vLLM crash |
| B | Task 3 raw prompt | 22 | ❌ Failed | Samsung Internet + vision-only suppressed tree → coordinate loop |
| C | Task 3 raw prompt | 23 | ⚠️ Partial | Email sent to Outbox (no connection); model tried to re-compose |
| D | Like first 5 X posts | 23 | ⚠️ 4/5 | Posts 1–4 liked cleanly; post 5 scroll loop → timeout |

### Customisations applied to base mobilerun framework

| Change | Reason |
|--------|--------|
| Stuck loop detection (3x same action → redirect + screenshot) | Prevent infinite repetition |
| Alternating A/B/A/B detection (6-step sliding window) | Catch loops the 3x detector misses |
| Vision always on (screenshot every step) | Apps that block accessibility tree are otherwise blind |
| Both tree + screenshot sent every step | Tree suppression broke Gmail; dual input handles all app types |
| `parallel_tools: false` | Parallel actions caused blind cascade failures in Gmail |
| Step budget injection per turn | Keeps model goal-focused near step limit |
| Human-like char-by-char typing (20–50ms delays) | Realism; harder to detect as automated |
| show_touches on ADB connect | Tap/swipe positions visible on screen |
| XML parser fixes | Local model emits slightly malformed XML; parser now handles it |
| Max steps 60 → 40 | Prevents runaway tasks; forces efficiency |
| System prompt: Chrome over Samsung Internet | Samsung Internet blocks accessibility tree on YouTube |
| System prompt: Gmail Outbox = success | Gmail Retry button opens blank compose, creating duplicates |
| Loop detector redirect: scroll → tap to open | Model was scrolling for Like button instead of opening post |

---

*Benchmark prepared July 14, 2026. Extended runs July 14–15, 2026. All tasks on the same Android device using a self-hosted vLLM GPU server on a private network.*
