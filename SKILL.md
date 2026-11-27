---
name: rubydai-swift-ui-bonus-skill
description: Hard-won runtime checklist for any SwiftUI / iOS Swift app — silent bugs that build fine but crash or misbehave at runtime, launch-time timing traps (ATT/permission prompts, modals at launch), first-use latency (slow first sheet, DNS/TLS, first WKWebView, full-screen loaders), phone heating / battery drain while idle (repeatForever backgrounds under blur, timers, looping web views), how to measure it on the simulator, and how to cache the APIs the app calls (client RAM/disk, URLCache, Cloudflare Cache API, KV/R2/Redis). Use whenever working in a SwiftUI or Xcode project — writing, porting or reusing views, adding services/networking/audio/AdMob ads (banners, interstitials, the ATT prompt)/permissions, touching app launch, debugging "it's slow the first time", "the phone gets hot" or "nothing happens when I tap", adding caching to a backend or TTS/AI/API call, or preparing a release — even if the user doesn't ask for a checklist.
---

# SwiftUI bonus skill: silent bugs, first-use latency, API caching

"Build succeeded" is not "works". This skill holds the lessons that compilers and quick tests miss. The full guides are in `references/`; read the relevant one before acting, and quote its commands rather than re-deriving them.

| Situation | Read |
|---|---|
| Porting/reusing a view, adding a service, touching launch flow, permissions, audio, concurrency, localization, or doing a pre-release pass (incl. §17 export compliance and §18 App Store languages before uploading a build) | `references/silent-bugs-checklist.md` |
| A screen is slow the first time, the app "feels laggy", or you need to measure timing on the simulator | `references/silent-bugs-checklist.md` §13–14 |
| The phone gets hot or the battery drains, even when the app is idle | `references/silent-bugs-checklist.md` §15 |
| Adding or reviewing caching for an API, backend, TTS/AI/third-party call, or media downloads | `references/api-caching-guide.md` |
| Adding AdMob: SDK setup, the ATT prompt, banner size/placement, interstitials, testing ads; or an app for kids (no tracking, parental gate, §6) | `references/admob-guide.md` |

## Always-on rules (apply without being asked)

**After any change**
- Build for the simulator and read the *warnings* of the touched files (§0): in Swift 5 mode with `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`, isolation bugs are only warnings.
- Check runtime contracts, not just compilation: environment injection on every entry path, `NS…UsageDescription` keys, `AVAudioSession` category before each play, double-tap re-entrancy, errors surfaced to the UI.

**UI text**
- Keep on-screen copy short: labels of a few words, hints and footers one short sentence, no explanations of how things work behind the scenes. Write the action or the fact, not the reasons; a confirmation says what is lost and "This can't be undone", nothing more. Long explanations belong in a help page, never on the screen. Shorten every locale together.

**Launch flow**
- System prompts (ATT etc.) are silently ignored unless the app is `.active` — wait for `didBecomeActiveNotification` explicitly; never rely on another step's delay.
- Don't present your own modal while launch prompts are pending, and don't ask on the very first launch if a system prompt is already showing.

**Release**
- A multilingual app must ship one `.lproj` per language plus `CFBundleLocalizations`, or the App Store lists only some of its languages (§18). Check `ls <App>.app/*.lproj` on the built app.
- Every app needs `ITSAppUsesNonExemptEncryption` = `NO` in Info.plist (or `INFOPLIST_KEY_ITSAppUsesNonExemptEncryption`) when it only uses HTTPS / Keychain / OS crypto, or each uploaded build shows "Missing Compliance". Ask before writing `NO` if the app ships its own encryption (§17).

**Ads (AdMob)**
- Ask ATT right at launch once the app is active (App Review must see it), then start the SDK; no ad loads before the answer.
- Debug = Google test units, Release = real units, chosen by the build configuration (`#if DEBUG`), never by a flag someone must remember. Add a Release-only Run Script build phase that **fails the Archive** when the built app still holds test units or the test App ID (`references/admob-guide.md` §1.1); a wrapper app (Capacitor, WebView, React Native) asks the native side for `#if DEBUG` through a plugin to pick the units, and the build phase checks the bundled JS for the real units (and no local API) instead.
- AdMob "Test devices" on iOS match by IDFA, so they work only after tapping **Allow** on the ATT prompt; check for the "Test mode" label before tapping any ad.
- Banners: full width with `inlineAdaptiveBanner(width:maxHeight:)` (anchored is deprecated, "large anchored" is ~125 pt), zero height until loaded, never next to frequently tapped buttons. A `UIViewRepresentable` list doesn't see SwiftUI's `safeAreaInset`: pass the banner height as its `contentInset.top`.
- Interstitials: preload, cap frequency (persisted), present from the top-most view controller, and make memory-heavy work wait until the ad has closed.

**First-use latency** (measure first, then fix in this order)
1. Never cover data you already have with a full-screen loader; show it and use a small non-blocking indicator.
2. Warm DNS + TLS to the hosts the key screens call, through the *same* `URLSession` each service uses.
3. Warm WebKit once (hidden `WKWebView`, kept alive for the session).
4. Prefetch likely next taps (debounced) with in-flight dedupe so the detail screen awaits the same `Task`.
5. Uninstall the app before each measurement (`URLCache` fakes good numbers); treat simulator numbers as a lower bound and confirm on a device with a Release build.

**Idle cost (heat, battery)**
- Nothing may animate or tick forever on shared/background views: no `repeatForever` background gradient (it re-renders every `.ultraThinMaterial` blur on top of it each frame), no global timers or looping web views. Keep them inside the one screen that needs them and stop them in `onDisappear`.
- Verify idle cost: with the app sitting still, both the app process *and* `backboardd` (render server) should be near 0% CPU on the simulator.

**API caching**
- Client first (RAM + disk with LRU, file I/O off the main actor), then the edge (Cloudflare Cache API, per data center, not on `*.workers.dev`, write with `ctx.waitUntil`), persistent stores only when measurements justify them.
- Key = every result-affecting parameter, normalized, hashed, with a version prefix. Never cache errors or per-user/private data in shared layers.
- Keep the response format unchanged so shipped app versions benefit; cap input size and rate-limit public endpoints; verify MISS → HIT with identical bodies before deploying.

## Temporary instrumentation

When you add timing probes or debug launch arguments to investigate, mark every line (e.g. `// PERF-TMP`) and remove all of them before finishing; grep for the marker to confirm.

## Keeping this skill current

The references are copies. When a project's own `docs/silent-bugs-checklist.md` or `docs/api-caching-guide.md` is newer, prefer it, and offer to sync the improvements back into this skill.
