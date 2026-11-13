# AdMob in a SwiftUI app: banners, interstitials, ATT

Google Mobile Ads SDK 13 (Swift names: `MobileAds`, `BannerView`, `InterstitialAd`, `Request`). Everything here builds fine when done wrong. The failures show up in App Review, on a real device, or as layout bugs only visible with a loaded ad.

## 1. Setup

- **Swift Package Manager:** `https://github.com/googleads/swift-package-manager-google-mobile-ads`, product `GoogleMobileAds`. Pin a range such as `13.x..<14.0.0`. Check the latest tag with
  `git ls-remote --tags https://github.com/googleads/swift-package-manager-google-mobile-ads | awk -F/ '{print $3}' | sort -V | tail -3`.
- **Info.plist:**
  - `GADApplicationIdentifier`: the AdMob **app** ID (`ca-app-pub-…~…`). Use the real one in every configuration, Debug included.
  - `SKAdNetworkItems`: Google's list (about 50 identifiers). Copy it from another app of the same account or from Google's docs; don't trim it.
  - `NSUserTrackingUsageDescription` (or `INFOPLIST_KEY_NSUserTrackingUsageDescription`): an honest sentence. Saying what is *not* used for ads (e.g. "Your messages are never used for ads") reads well to users and reviewers.
- **Ad unit IDs:** keep them in one place. Debug builds use Google's test units, Release the real ones:

  ```swift
  enum Ads {
      #if DEBUG
      static let bannerUnitID = "ca-app-pub-3940256099942544/2934735716"        // Google test banner
      static let interstitialUnitID = "ca-app-pub-3940256099942544/4411468910"  // Google test interstitial
      #else
      static let bannerUnitID = "ca-app-pub-XXXXXXXXXXXXXXXX/NNNNNNNNNN"
      static let interstitialUnitID = "ca-app-pub-XXXXXXXXXXXXXXXX/NNNNNNNNNN"
      #endif
  }
  ```

  Tapping your own real ads while developing can get the AdMob account flagged. After a Release build, confirm that the binary holds only the real units:
  `strings "<App>.app/<App>" | grep ca-app-pub- | sort -u`.
- **app-ads.txt** on the developer website (the domain set as the app's Marketing/Support URL): `google.com, pub-XXXXXXXXXXXXXXXX, DIRECT, f08c47fec0942fa0`. The publisher ID is per AdMob account, so apps of the same account share it.
- **UMP (Google's GDPR consent form):** optional. If the owner accepts limited, non-personalized ads in the EEA/UK/Switzerland, skip it entirely, and say so in the privacy policy.

## 2. The tracking prompt (ATT)

- **Ask right at launch**, as soon as the app is active. App Review opens the app and expects to see the prompt; a prompt that only shows after some later step (e.g. after a first download or sign-in) often leads to a rejection ("we were unable to locate the App Tracking Transparency permission request").
- It is silently ignored unless the app is `.active` (see silent-bugs §13):

  ```swift
  if UIApplication.shared.applicationState != .active {
      for await _ in NotificationCenter.default.notifications(named: UIApplication.didBecomeActiveNotification) { break }
  }
  let status = await ATTrackingManager.requestTrackingAuthorization()
  ```

- Start the SDK (`await MobileAds.shared.start()`) **after** the user has answered, so nothing loads an ad before the prompt. Banners render only once a "can show ads" flag is set after `start()`.
- A paid "remove ads" purchase can skip both the prompt and the SDK.
- In App Store Connect → App Privacy, declare the ad data **as used for tracking**, consistent with the prompt.

## 3. Banners

- **Size:** a fixed 320×50 looks like a floating box on wider phones and leaves dark edges. Use the full width.
  - `currentOrientationAnchoredAdaptiveBanner(width:)` is **deprecated** in SDK 13.
  - Its replacement `largeAnchoredAdaptiveBanner(width:)` is about **125 pt tall** on a ~400 pt wide phone, too much for a chat or list screen.
  - Use `inlineAdaptiveBanner(width:maxHeight:)` with `maxHeight` ≈ 50–60. It's not deprecated, fills the width, and the height of the ad that actually loaded is `bannerView.adSize.size.height` in `bannerViewDidReceiveAd`.
- **Measure the width** (`onGeometryChange`), create the banner only once the width is known, and rebuild it (`.id(Int(width))`) when the width changes (rotation, iPad sidebar).
- **Take no space until loaded:** height 0 before `bannerViewDidReceiveAd`, and again after a failure (offline, no fill). Put the `BannerView` at the top of a plain container `UIView` so the SwiftUI frame can be the maximum height while the ad is shorter.
- Leave `rootViewController` nil; the SDK presents the landing page from the banner's window.
- **Placement:** never next to buttons the user taps often (composer, send, tab bar). Accidental taps break AdMob policy. Under the navigation bar or at the bottom of a list screen is fine. Hide a top banner while the keyboard is up.
- **A list that scrolls under a transparent navigation bar** (`.ignoresSafeArea(edges: .top)`):
  - Apply `.safeAreaInset(edge: .top) { banner }` *after* `.ignoresSafeArea`. Applied to the inner view instead, it lands under the status bar.
  - A `UITableView`/`UICollectionView` wrapped in `UIViewRepresentable` does **not** see SwiftUI's `safeAreaInset`, so the first rows hide behind the banner. Measure the banner strip's height and pass it in as `contentInset.top` (and `verticalScrollIndicatorInsets.top`).
  - Give the banner strip a solid background so rows scrolling under it don't show through.
- Dark bars around Google's **test** banner come from the test creative itself, not from your layout.

## 4. Interstitials

- **Preload** after the SDK starts and again after each dismissal. A loaded ad expires after one hour, so reload when it's older than ~55 minutes.
- **Show at natural breaks** (a download starting, switching modes, finishing a task), never in the middle of typing or reading. Cap the frequency: persist the last show time in `UserDefaults` so the cap survives relaunches (e.g. at most one ad per 10 minutes for the frequent trigger).
- **Present from the top-most view controller**, including presented sheets:

  ```swift
  var top = keyWindow?.rootViewController
  while let presented = top?.presentedViewController, !presented.isBeingDismissed { top = presented }
  ad.present(from: top)
  ```

  Wait ~0.3–0.4 s when the trigger is a `Menu` item or a button that closes a sheet; presenting during that animation fails.
- **Set a "full-screen ad is up" flag synchronously** when you decide to show, before that delay, and clear it in `adDidDismissFullScreenContent` *and* in `didFailToPresentFullScreenContentWithError`. Anything memory-heavy (loading an on-device ML model, decoding large media) should `await` a "wait until the ad closes" helper backed by that flag. The ad's web view and video use a lot of memory, and on a 4 GB phone the two together can get the app killed.
- Keep the `FullScreenContentDelegate` in a small separate `@MainActor` `NSObject`, held strongly by the service, so the service doesn't have to inherit from `NSObject`.
- On first presentation, the interstitial can take a few seconds to appear while its web view starts.

## 5. Testing

- On the simulator, killing the app while the ATT prompt is open leaves the dialog **lingering** on the next launches, even when that launch never asked. Read the log (`[ATTrackingManager] requestTrackingAuthorization…`) rather than trusting a screenshot. Reboot the simulator (`simctl shutdown` + `boot`) to clear it.
- A headless simulator can't tap the prompt. To get past it, add a temporary launch argument (`-skipATT`, marked `// ADS-TMP`) that skips the request, plus another one that calls the interstitial a few seconds after start. Remove them all before finishing and grep for the marker.
- Log the loaded banner size (`bannerView.adSize`) once to check the real height.
- On a real device, test: the prompt at a fresh install (delete the app first), both answers, a banner on every screen that has one, an interstitial at each trigger, closing it, and then the memory-heavy action right after.
