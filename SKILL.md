---
name: binh-swift-ui-bonus-skill
description: Hard-won runtime checklist for any SwiftUI / iOS Swift app — silent bugs that build fine but crash or misbehave at runtime, launch-time timing traps (ATT/permission prompts, modals at launch), first-use latency (slow first sheet, DNS/TLS, first WKWebView, full-screen loaders), how to measure it on the simulator, and how to cache the APIs the app calls (client RAM/disk, URLCache, Cloudflare Cache API, KV/R2/Redis). Use whenever working in a SwiftUI or Xcode project — writing, porting or reusing views, adding services/networking/audio/ads/permissions, touching app launch, debugging "it's slow the first time" or "nothing happens when I tap", adding caching to a backend or TTS/AI/API call, or preparing a release — even if the user doesn't ask for a checklist.
---

# SwiftUI bonus skill: silent bugs, first-use latency, API caching

"Build succeeded" is not "works". This skill holds the lessons that compilers and quick tests miss. The full guides are in `references/` (written in Vietnamese); read the relevant one before acting, and quote its commands rather than re-deriving them.

| Situation | Read |
|---|---|
| Porting/reusing a view, adding a service, touching launch flow, permissions, audio, concurrency, localization, or doing a pre-release pass | `references/silent-bugs-checklist.md` |
| A screen is slow the first time, the app "feels laggy", or you need to measure timing on the simulator | `references/silent-bugs-checklist.md` §13–14 |
| Adding or reviewing caching for an API, backend, TTS/AI/third-party call, or media downloads | `references/api-caching-guide.md` |

## Always-on rules (apply without being asked)

**After any change**
- Build for the simulator and read the *warnings* of the touched files (§0): in Swift 5 mode with `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`, isolation bugs are only warnings.
- Check runtime contracts, not just compilation: environment injection on every entry path, `NS…UsageDescription` keys, `AVAudioSession` category before each play, double-tap re-entrancy, errors surfaced to the UI.

**Launch flow**
- System prompts (ATT etc.) are silently ignored unless the app is `.active` — wait for `didBecomeActiveNotification` explicitly; never rely on another step's delay.
- Don't present your own modal while launch prompts are pending, and don't ask on the very first launch if a system prompt is already showing.

**First-use latency** (measure first, then fix in this order)
1. Never cover data you already have with a full-screen loader; show it and use a small non-blocking indicator.
2. Warm DNS + TLS to the hosts the key screens call, through the *same* `URLSession` each service uses.
3. Warm WebKit once (hidden `WKWebView`, kept alive for the session).
4. Prefetch likely next taps (debounced) with in-flight dedupe so the detail screen awaits the same `Task`.
5. Uninstall the app before each measurement (`URLCache` fakes good numbers); treat simulator numbers as a lower bound and confirm on a device with a Release build.

**API caching**
- Client first (RAM + disk with LRU, file I/O off the main actor), then the edge (Cloudflare Cache API, per data center, not on `*.workers.dev`, write with `ctx.waitUntil`), persistent stores only when measurements justify them.
- Key = every result-affecting parameter, normalized, hashed, with a version prefix. Never cache errors or per-user/private data in shared layers.
- Keep the response format unchanged so shipped app versions benefit; cap input size and rate-limit public endpoints; verify MISS → HIT with identical bodies before deploying.

## Temporary instrumentation

When you add timing probes or debug launch arguments to investigate, mark every line (e.g. `// PERF-TMP`) and remove all of them before finishing; grep for the marker to confirm.

## Keeping this skill current

The references are copies. When a project's own `docs/silent-bugs-checklist.md` or `docs/api-caching-guide.md` is newer, prefer it, and offer to sync the improvements back into this skill.
