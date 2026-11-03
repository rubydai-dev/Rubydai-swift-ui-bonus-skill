# Silent-bug checklist for SwiftUI projects

This guide works for any SwiftUI project. Copy it into another project and change only the examples.

A "silent bug" is one where **the build succeeds, but the app crashes or misbehaves at runtime**. There are two kinds:

- **Crash:** the app is killed as soon as the user walks into that exact flow.
- **Quietly wrong:** the app doesn't crash, but a button does nothing, an error is swallowed, the wrong data is sent to the server, or the sound gets cut.

The compiler catches neither. So "build OK" after each change is **not enough**.

---

## When to run the checklist

- After porting or copying a view from another project. This is when bugs are most likely, because the view carries "unwritten contracts" with its old parent.
- When reusing a component somewhere new, e.g. a renderer that used to live only inside sheet A is now placed on screen B.
- Before a release.

Order of work:

1. Get the list of warnings (section 0).
2. Sweep each group from section 1 to section 15.
3. Try the manual scenarios in section 16.

---

## 0. Read the warnings of a clean build

Many runtime bugs still show up as **warnings**, especially when the project is in Swift 5 mode with `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor` turned on. Isolation violations are then only warnings.

```sh
# touch to force a full recompile; otherwise Xcode only prints warnings for files that just changed
find "<App>" -name '*.swift' -exec touch {} +
xcodebuild -workspace "<App>.xcworkspace" -scheme "<App>" \
  -destination 'generic/platform=iOS Simulator' build 2>&1 \
  | grep -E "\.swift:[0-9]+:[0-9]+: warning" | grep -v "/Pods/" | sort -u
```

Read these warnings carefully:

| Warning | Meaning |
|---|---|
| `main actor-isolated ... cannot be called from outside of the actor` | `@MainActor` code is running on a background thread (usually inside `Task.detached`). |
| `expression is 'async' but is not marked with 'await'` | The compiler inserts a hop back to the main actor, so `Task.detached` has no effect and the main thread is still blocked. |
| `capture of ... with non-Sendable type` | A possible data race. |
| `result of 'try?' is unused` | An error is swallowed. Decide whether it's deliberate. |

---

## 1. Dependencies from the environment

**Crash:** `No ObservableObject of type X found` or `No Observable object of type X found`.

```sh
grep -rn "@EnvironmentObject" --include='*.swift' .
grep -rhoE "@Environment\([A-Z][A-Za-z]+\.self\)" --include='*.swift' . | sort | uniq -c
grep -rnE "\.environmentObject\(|\.environment\([a-zA-Z]" --include='*.swift' .
```

- For each type, find where it's injected. If it's injected at the root `App`, every view including sheets and fullScreenCovers receives it. If it's injected in a child view, every place that reuses that component must inject it too.
- **The environment does not pass through** a `UIHostingController` you create yourself, `ImageRenderer`, or a `UIViewControllerRepresentable` hosting SwiftUI. If a view is rendered in one of these, pass the data as parameters.
- **The root fix:** for optional dependencies, don't use `@EnvironmentObject`. Use an optional environment value, so forgetting to inject only loses the feature instead of crashing:

  ```swift
  extension EnvironmentValues {
      @Entry var audioPlayer: AudioPlayerController?
  }
  // child view
  @Environment(\.audioPlayer) private var controller
  var body: some View { if let controller { SpeakerButton(controller: controller) } }
  // parent view
  .environment(\.audioPlayer, player)
  ```

## 2. Permission keys in Info.plist

**Crash:** iOS kills the app the moment it calls an API that needs a permission and the `NS...UsageDescription` key is missing. It only happens on the very first call to that API, so it slips through easily.

| API / action | Required key |
|---|---|
| `UIImagePickerController` camera, `AVCaptureDevice` | `NSCameraUsageDescription` |
| `AVAudioEngine`, `AVAudioRecorder`, `requestRecordPermission` | `NSMicrophoneUsageDescription` |
| `SFSpeechRecognizer` | `NSSpeechRecognitionUsageDescription` |
| `PHPhotoLibrary` reading the library | `NSPhotoLibraryUsageDescription` (`PhotosPicker` doesn't need it) |
| **`UIActivityViewController` / `ShareLink` sharing an image** (the user taps "Save Image"), `UIImageWriteToSavedPhotosAlbum` | **`NSPhotoLibraryAddUsageDescription`** (the most often forgotten) |
| `ATTrackingManager` | `NSUserTrackingUsageDescription` |
| `CLLocationManager` | `NSLocationWhenInUseUsageDescription` |

```sh
grep -rnE "UIImagePickerController|AVCaptureDevice|AVAudioEngine|SFSpeechRecognizer|PHPhotoLibrary|UIActivityViewController|ShareLink|UIImageWriteToSavedPhotosAlbum|ATTrackingManager|CLLocationManager|CNContactStore|EKEventStore" --include='*.swift' .
grep -o "NS[A-Za-z]*UsageDescription" *.xcodeproj/project.pbxproj */Info.plist 2>/dev/null | sort -u
```

## 3. Operations that can trap

```sh
grep -rnE "try!|fatalError|preconditionFailure|as! " --include='*.swift' .
grep -rnE "[A-Za-z0-9_\)\]]![^=]" --include='*.swift' . | grep -v '"'      # force unwrap
grep -rn "uniqueKeysWithValues" --include='*.swift' .                    # crashes on duplicate keys
grep -rnE "\.removeFirst\(|\.removeLast\(|\.first!|\.last!|randomElement\(\)!" --include='*.swift' .
```

- **Array indexes:** `arr[i]` needs a guard, `indices.contains(i)` or `i < count`. Be especially careful with indexes stored in `@State` when the source data changes.
- **Ranges:** `a..<b` and `a...b` crash when `a > b`. `Slider(in: 0...x)` crashes when `x` is NaN. Note that `max(NaN, 0.01)` returns NaN.
- **Int from Double:** `Int(x)` crashes when `x` is NaN or infinite. Audio durations, 0/0 divisions and progress values can be NaN, so check `isFinite` first.
- **`abs(Int.min)`** overflows. Don't use `abs(hashValue) % n`.
- **`Dictionary(uniqueKeysWithValues:)`** crashes on duplicate keys. For data from a server, use `Dictionary(_:uniquingKeysWith:)`.
- **Parsers for user-supplied files** (zip, epub, docx, binary): every `subdata` or `data[i]` must be bounds-checked against `data.count`. Truncated or corrupt files are normal.

## 4. `String(format:)` with translated strings

**Crash (`EXC_BAD_ACCESS`)** when a translation has different specifiers from the arguments, e.g. `%@` but an `Int` is passed, or two specifiers with only one argument. It only shows when the device uses that exact language.

Script: for every key used in `String(format: strings.KEY, ...)`, compare the specifier set of **every** translation with the English one.

```python
import re, glob, collections, subprocess
keys = set(re.findall(r"\.(\w+)$", subprocess.run(["bash","-c",
  "grep -rhoE 'String\\(format: *[a-zA-Z.]+\\.[a-zA-Z0-9_]+' --include='*.swift' ."],
  capture_output=True, text=True).stdout, re.M)) - {"current"}
spec = re.compile(r"%(?:(\d+)\$)?[-+ #0]*\d*(?:\.\d+)?(lld|ld|d|i|u|@|f|s)")
norm = lambda t: {"lld":"d","ld":"d","i":"d","u":"d"}.get(t, t)
def specs(s):
    return tuple(sorted((int(m[1]) if m[1] else i+1, norm(m[2]))
                        for i, m in enumerate(spec.finditer(s.replace("%%", "")))))
res = collections.defaultdict(set)
for f in glob.glob("**/*.swift", recursive=True):
    src = open(f, encoding="utf-8").read()
    for k in keys:
        for m in re.finditer(r'\b' + k + r'\s*[:=]\s*"((?:[^"\\]|\\.)*)"', src):
            res[k].add(specs(m.group(1)))
for k, v in res.items():
    if len(v) > 1: print("MISMATCH", k, v)
```

Then compare the English specifiers with the argument types at the call site.

## 5. IDs in `ForEach`

`ForEach(xs, id: \.self)` with duplicate elements draws wrong, drops rows, or keeps the wrong state. Inside a `List` or a lazy stack, it can crash.

```sh
grep -rnE "ForEach\([^)]*id: \\\\\.self" --include='*.swift' . | grep -vE "\.\.<|allCases|indices"
```

- For data from a server or the user (answer choices, tags, sublists), use `ForEach(Array(xs.enumerated()), id: \.offset)`, or deduplicate if duplicates carry no meaning (e.g. repeated letters in a word).
- For `Identifiable` models from JOIN or FTS queries, check whether the database returns the same id more than once.

## 6. Handlers called twice (double tap, tapping during an await)

**Wrong data:** tapping Continue/Submit twice quickly skips an item, or the "finalize" step runs twice and posts the result to the server twice.

- Every async function called from a button that changes state or hits the network needs a guard flag (`guard !isSubmitting`). The flag must be set **before the first `await`**.
- Functions that grade or advance should take **the id of the item being answered** and ignore the call if that id is no longer the current item:

  ```swift
  func grade(itemId: Item.ID, correct: Bool) async {
      guard !isFinalizing, let item = currentItem, item.id == itemId else { return }
      ...
  }
  ```

## 7. Swallowed errors

- A controller has `@Published var errorMessage`, but the view that owns it doesn't attach `.toast(...)` or `.alert(...)`. The user taps and nothing visible happens.

  ```sh
  # files that create a controller (change "Controller()" to the project's naming convention) but never read errorMessage
  for f in $(grep -rl "Controller()" --include='*.swift' .); do
    grep -q errorMessage "$f" || echo "$f"; done
  ```

- A `catch` that only writes to `Logger` in a flow the user believes was saved, e.g. a summary screen shows "done" while the request failed. Report the error in the UI.
- `try?` on writes.
- `try rows.compactMap { try decode(...) }`: one bad row breaks the whole list. Consider skipping just the bad row and logging it.

- **A property's default value does NOT save `Decodable`.** `var mastered: Int = 0` still throws `keyNotFound`
  when the JSON lacks that key (the error reads "The data couldn't be read because it is missing."). This often happens
  when one model is used for several endpoints that each return a different set of keys — the request
  succeeds on the server but the UI reports an error. The fix: write `init(from:)` and use `decodeIfPresent`,
  or make the property optional:

  ```swift
  init(from decoder: Decoder) throws {
      let c = try decoder.container(keyedBy: CodingKeys.self)
      id = try c.decode(String.self, forKey: .id)
      mastered = try c.decodeIfPresent(Int.self, forKey: .mastered) ?? 0
  }
  ```

  ```sh
  # Decodable models with non-optional properties that have default values
  grep -rn -B30 "= 0$\|= \[\]$\|= false$" Models/ | grep -E "struct .*Decodable|struct .*Codable"
  ```

## 8. View lifecycle

- **Audio, timers, speech:** every view that owns a player or a timer needs `.onDisappear { stop() }`. Having the X button call `stop()` isn't enough, because the user can swipe back or swipe the sheet down.
- **`@State` or `@StateObject` initialized from a parameter** (`_x = State(initialValue: param)`) only takes the value the first time. If the view is reused with a new parameter, it shows stale data. Use `.id(param)` or `.onChange(of: param)` to reset.
- When a child view resets its state in `.onChange(of: id)`, the body still renders **once with the new data and the old state** before onChange runs. Index-like state pointing into the new data can go out of bounds.
- **State that's never used:** a script that finds `@State` that is assigned but has no `$x` binding and is never read, i.e. a button that does nothing:

  ```python
  import re, glob
  for f in glob.glob("**/*.swift", recursive=True):
      s = open(f, encoding="utf-8").read()
      for m in re.finditer(r"@State (?:private )?var (\w+)", s):
          n = m.group(1)
          uses = len(re.findall(r"(?<![\w.])\$?" + n + r"\b", s)) - 1   # minus the declaration
          binds = len(re.findall(r"\$" + n + r"\b", s))
          assigns = len(re.findall(r"(?<![\w$])(?:self\.)?" + n +
              r"\s*(?:=[^=]|\.toggle\(\)|\+=|-=|\.append|\.insert|\.remove)", s)) - 1
          if binds == 0 and uses - assigns - binds <= 0: print(f, n)
  ```

## 9. Audio (`AVAudioSession` is shared by the whole app)

- There is **only one session for the whole app**. Any module that changes the category (e.g. `.record` for speech recognition) affects every other module. Each player must set the category again **right before every `play()`**, not just once in a singleton's `init`.
- `setActive(false)` **stops every player running** on the session, including another module's background music. Only deactivate the session when you're sure nothing else is playing.
- `AVAudioEngine.inputNode.installTap`: if `outputFormat(forBus: 0)` has `sampleRate == 0` or `channelCount == 0` (no mic, or the mic is busy with a call), it throws an **NSException** and the app crashes. Guard before calling, and `removeTap` if `engine.start()` fails.

```sh
grep -rn "setCategory\|setActive\|installTap" --include='*.swift' .
```

## 10. Concurrency

- With `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`, every type is `@MainActor` by default. Code running in `Task.detached` or `DispatchQueue.global` that calls into those types violates isolation. In Swift 5 mode, the compiler only warns (see section 0).
- Pure data-processing types (parsers, tokenizers, zip readers) meant to run in the background should be marked `nonisolated` at the type or function level.
- `LocalizedError` types whose `errorDescription` reads main-actor localized strings: keep the error type at the default isolation and mark only the throwing functions `nonisolated`.
- Check at runtime with the environment variable `SWIFT_UNEXPECTED_EXECUTOR_LOG_LEVEL=2`: the wrong executor crashes; `1` only logs. Also turn on **Thread Sanitizer** and **Main Thread Checker** in Scheme → Diagnostics.

## 11. Navigation and presentation silently ignored

- A `NavigationStack` nested inside a view that was already pushed, or a `navigationDestination` placed outside the stack or inside a `List`/lazy container: SwiftUI only logs and ignores it.
- Opening a sheet or fullScreenCover while another one is presenting or dismissing: the second one is ignored. Wait about 350 ms or use `onDismiss`.
- Switching back and forth between two `NavigationStack`s (e.g. a signed-in branch and a guest branch) can crash UIKit's navbar. Only one branch should own the stack.

```sh
grep -rn "NavigationStack\|navigationDestination" --include='*.swift' .
```

## 12. Locale and language

- `Locale.preferredLanguages` returns codes with scripts: `zh-Hant-TW`, `zh-Hans-CN`, `pt-BR`, `fil-PH`. Matching on the `"zh"` prefix sends Traditional Chinese users to Simplified. Handle `zh-Hant*` first.
- When adding a language, check plurals, language names (`Locale.localizedString(forIdentifier:)`) and format specifiers (section 4).
- If the app switches language in-app (independent of the device language), every change should go through **one function** in the store that holds the language, and that function assigns the value **without animation**:

  ```swift
  func apply(_ language: AppLanguage?) {
      var transaction = Transaction(animation: nil)
      transaction.disablesAnimations = true
      withTransaction(transaction) { selectedLanguage = language }
  }
  ```

  Don't assign the language inside `withAnimation`, and don't rely on `.animation(nil, value: languageCode)` on a parent view. Views read the store directly through Observation (many live in their own `NavigationStack`), so a modifier on the parent has no effect. If the change lands in an animated transaction (restoring a session while scrolling, closing a menu on sign-out), text cross-fades between the two languages.
- A `static var currentCode` copy of the language (for code that can't reach the environment) is **not observed**. Use it only for errors, services and accessibility labels; in `body`, read the language from the store through the environment so views update when it changes.

## 13. Timing at app launch

Launch steps often **silently depend on each other's timing**. Removing or reordering one step can quietly break a later one.

- **System permission prompts are ignored while the app isn't active.** For example, `ATTrackingManager.requestTrackingAuthorization` called before the app is `.active` returns immediately without showing anything. The code may have worked before only because another step (a network request, a consent form) happened to add enough delay. Remove that step and the prompt disappears. Wait for the app to become active explicitly:

  ```swift
  if UIApplication.shared.applicationState != .active {
      for await _ in NotificationCenter.default.notifications(named: UIApplication.didBecomeActiveNotification) { break }
  }
  ```

- **The app's own modals compete with system dialogs.** Don't open your own sheet or modal while the launch-time "ask the user" steps (consent form, ATT, notifications) haven't finished. Wait for a "launch flow finished" flag, and for the session state too if the modal's content depends on it (e.g. the account's language).
- **Don't ask the user anything on the very first launch** if another dialog is already showing. If the modal has a "Later" button, also handle the user swiping it away (e.g. start the cooldown as soon as the modal appears).
- On the simulator, killing the app while a system dialog is open can leave that dialog **lingering** on the next launch. Read the log to know whether this launch really called the permission API; don't rely on screenshots alone.

## 14. First-use latency

No crash, no wrong data, but the **first** time a screen opens it takes 2–4 seconds, while later times are fast. Few people notice during development, because the dev device already has warm caches and connections. For a lookup app, one such experience is enough for a user to delete it.

**Common causes, which usually add up:**

| Cause | Sign |
|---|---|
| The first connection to each host: DNS + TLS (0.3–1 s per host on 4G) | The first request is slow, later ones are fast |
| The backend database lives in one fixed region (even if the Worker or edge function has no cold start) | Items not yet in the edge cache are slower than cached ones |
| The first `WKWebView` has to start the WebKit process (a few hundred ms on the main thread) | Screens with a web view (SVG, HTML, ad banners) stutter the first time |
| SwiftUI's first layout and the Swift runtime's first conformance lookups (much heavier in Debug) | The main thread stalls for a few hundred ms when a sheet opens the first time |
| **A full-screen loader covering data you already have** | The user stares at an empty screen even though the app has a usable version of the data (e.g. the untranslated original) |

**Fixes, in order of impact:**

1. **Don't cover data you already have.** Show what you have right away (the original, an older cached copy), signal loading with a small non-blocking indicator, and swap in the new data when it arrives. This is the biggest change in how the app feels.
2. **Warm up connections** right after launch (wait 1–2 seconds for the first screen to settle): send a light `HEAD` request to every host the key screens will call.
   - **The connection pool belongs to each `URLSession`.** Warming through `URLSession.shared` does nothing for a service with its own `URLSession`. Warm through the exact session that service uses.
3. **Warm up WebKit:** create a hidden `WKWebView` (`loadHTMLString("")`) and **keep it alive for the whole session**, so the web content process is always running.
4. **Prefetch** data for what the user is likely to tap next (e.g. the first few search results), with a debounce because the list changes with every keystroke.
   - Add **in-flight dedupe**: store the running `Task` by key. If the detail screen opens while the prefetch is still running, `await` that same `Task` instead of sending a second request.
   - Run the prefetch `Task` unstructured, so the result still lands in the cache when the debounce cancels it.
   - If one response returns several items (e.g. every entry for a search term), cache all of them.
5. **Offline data** (downloadable packs, a bundled database) removes the network step entirely. Actively invite users to download the pack instead of leaving it behind a small icon.

**How to measure** (measure before fixing; don't guess):

- **Timestamp** with `os_log` at: the tap, `onAppear`, request start/end, web view creation start/end. Add a **main-thread heartbeat** (a 20 ms timer that logs when the gap between two ticks exceeds 80 ms) to catch main-thread stalls. Use a shared marker (e.g. `// PERF-TMP`) so everything can be removed cleanly after measuring.

  ```swift
  nonisolated enum PerfTmp {
      nonisolated(unsafe) static var t0 = CACurrentMediaTime()
      static let log = Logger(subsystem: "perf.tmp", category: "PerfTmp")
      static func mark(_ s: String) {
          log.notice("PERF \(Int((CACurrentMediaTime() - t0) * 1000), privacy: .public)ms \(s, privacy: .public)")
      }
  }
  ```

- **Open the screen being measured automatically** with a measurement-only launch argument (e.g. `-PerfDetail`): wait a few seconds after launch, open the screen once, close it, then open it a second time with **different data**, to separate the "first time" cost from the "every time" cost.
- **Uninstall the app before every measurement**, because the previous run's `URLCache` makes the numbers look falsely good.

  ```sh
  xcrun simctl uninstall $DEV $BUNDLE && xcrun simctl install $DEV "$APP"
  xcrun simctl launch $DEV $BUNDLE -PerfDetail; sleep 20
  xcrun simctl spawn $DEV log show --last 30s --predicate 'subsystem == "perf.tmp"' --style compact
  ```

- **Time Profiler on the simulator:** attach with the Mac-side pid *and* the simulator's UDID, then export to XML to read the heavy frames on the main thread.

  ```sh
  PID=$(pgrep -f "<App>.app/<App>" | head -1)
  xcrun xctrace record --template 'Time Profiler' --device $DEV --attach $PID --time-limit 6s --output tp.trace
  xcrun xctrace export --input tp.trace \
    --xpath '/trace-toc/run[@number="1"]/data/table[@schema="time-profile"]' --output tp.xml
  ```

- **Measure the server separately** with `curl -w "dns=%{time_namelookup} tls=%{time_appconnect} ttfb=%{time_starttransfer}"` to separate connection cost, query time and the edge cache (look at the `cf-cache-status` and `age` headers).
- Simulator numbers are a **lower bound**. A real device on 4G with a slower CPU is much slower, so always try again on a device with a Release build.

## 15. The phone gets hot and the battery drains while the app sits idle

The app works, but the phone warms up even though the user is only reading. The cause is almost always **something redrawing the screen or waking the CPU constantly**.

**The most common culprit: an endlessly animating background under glass.** A background gradient using `withAnimation(...repeatForever...)` redraws the whole screen every frame. If there's `.ultraThinMaterial` or a blur on top, the GPU recomputes the blur every frame, on every screen that uses that background, even while the app is idle. Keep the background static. If you really want motion, limit it to one screen and turn it off in Low Power Mode or with Reduce Motion.

Other sources to sweep:

```sh
grep -rn "repeatForever\|TimelineView\|CADisplayLink" --include='*.swift' .
grep -rn "Timer\.\(scheduled\|publish\)\|Timer(" --include='*.swift' .
grep -rn "while true\|while !Task.isCancelled" --include='*.swift' .
grep -rn "setInterval\|requestAnimationFrame\|animation-iteration-count: *infinite" --include='*.swift' --include='*.html' --include='*.js' .
```

- Endless animations and timers should exist **only in the screen that needs them**, and stop in `onDisappear`. Don't put them on a background view shared by many screens.
- A `WKWebView` running a looping animation (`setInterval`, CSS `infinite`): each web view is its own CPU cost. A grid of many looping web views heats up very quickly.
- A `while` loop with `Task.sleep` needs an exit condition and must be cancelled when leaving the screen.
- An `onChange(of: x)` that reassigns `x` must converge, i.e. after the assignment the condition no longer holds. Otherwise it loops forever. A script can sweep for it: find `onChange(of: x)` closures that contain an assignment `x = …`, then read each one.

**Measuring on the simulator:** leave the app idle on a screen, then measure the CPU of the app *and* of the render server. Drawing (blur, gradients) isn't counted in the app's CPU; it shows up in `backboardd`.

```sh
for i in 1 2 3 4 5; do
  ps -Ao %cpu,comm | grep -E "backboardd|<App>" | grep -v grep
  sleep 1
done
```

When idle, both must be close to **0%**. If `backboardd` is high (tens of percent), something is redrawing constantly. Disable suspects one at a time (temporarily, marked so they can be restored) and measure again to pinpoint it.

On a device, use Xcode → Debug Navigator → Energy Impact, or Instruments (Time Profiler, Animation Hitches). Note: a Debug build with the debugger attached always runs hotter than a Release build.

## 16. Manual tests before a release

Scenarios where most silent bugs only show up when actually running:

| Scenario | Catches |
|---|---|
| Use every entry point to a feature, not just the main one (e.g. a practice screen opened from Home, from the favorites list and from another screen) | Missing environment injection |
| Turn off the network, or get rate-limited (429), then tap every button that hits the network | Swallowed errors |
| Deny each permission, then use that feature | The permission-denied path |
| Share an image → Save Image | Missing `NSPhotoLibraryAddUsageDescription` |
| Double-tap every Submit, Continue and Buy button quickly | Handlers called twice |
| Swipe a sheet down or swipe back while audio is playing | Missing stop in `onDisappear` |
| Switch between modules that use audio (record → play) | The session category |
| Switch the device language to each supported language, especially zh-Hant | Formats, locale |
| Device in English, sign in to an account with another language, force-quit the app, reopen and scroll immediately | Text cross-fading between two languages when the session restores |
| Import a corrupt or empty file (pdf, epub, docx, a renamed zip) | Parsers |
| Be on a call, or have another app hold the mic, then use the recording feature | `installTap` |
| Run a **Release** build on a real device | `#if DEBUG` branches, test ad units, localhost URLs |
| **Fresh install**, on 4G, open a key detail screen for the first time | First-use latency (section 14) |
| Fresh install, launch the app a first and a second time without tapping anything during launch | Ignored permission prompts, modals on top of system dialogs (section 13) |
| Leave the app idle for a few minutes on the main screens; feel the phone or watch Energy Impact | Something redrawing or running constantly (section 15) |
