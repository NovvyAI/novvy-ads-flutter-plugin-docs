The Flutter plugin for the Novvy Ads SDK.

## Features

- **Interstitial Ads** — Full-screen ads shown at natural transition points (e.g. between episodes)
- **Rewarded Ads** — Video ads that grant a reward when the user completes playback
- **Banner Ads** — 260×90 dp inline native views with server-driven show/hide scheduling
- **Feed Ads** — Native image or H5-zip ads inserted into scrollable lists with visibility-driven playback
- **AdMob Mediation** — Seamless Google AdMob mediation on both platforms
- **Zero lifecycle boilerplate** — The SDK manages load → show → dispose internally for all ad types
- **Centralized callbacks** — Register one `NovvyAdsCallback` at initialization; all ad events are delivered there

## Platform Support

| Platform | Minimum version | Notes |
|----------|----------------|-------|
| Android  | 7.0 (API 24+)  | |
| iOS      | 13.0+          | Simulator requires Apple Silicon Mac — see [iOS Notes](#ios-notes) |

## Installation

### 1. Add dependency

```yaml
dependencies:
  novvy_ads: 1.0.0-beta.41
```

```bash
flutter pub get
```

### 2. Configure Android

#### Add AdMob App ID

Open `android/app/src/main/AndroidManifest.xml` and add your AdMob App ID inside the `<application>` tag:

```xml
<manifest>
    <application>
        <meta-data
            android:name="com.google.android.gms.ads.APPLICATION_ID"
            android:value="ca-app-pub-xxxxxxxxxxxxxxxx~yyyyyyyyyy"/>
    </application>
</manifest>
```

> The app **crashes at launch** without this entry. Replace with your actual AdMob App ID.

#### Verify minSdk

```groovy
// android/app/build.gradle
android {
    defaultConfig {
        minSdk 24
    }
}
```

### 3. Configure iOS

#### Verify deployment target

```ruby
# ios/Podfile
platform :ios, '13.0'
```

#### Install pods

```bash
cd ios && pod install
```

The first `pod install` downloads the NovvyAds xcframework (~5 MB) automatically; subsequent runs skip the download unless the SDK version changes.

#### Add required keys to `Info.plist`

Open `ios/Runner/Info.plist` and add both keys inside the root `<dict>`:

**AdMob App ID** (required — app crashes at launch without it):

```xml
<key>GADApplicationIdentifier</key>
<string>ca-app-pub-xxxxxxxxxxxxxxxx~yyyyyyyyyy</string>
```

**App Tracking Transparency usage description** (required for iOS 14+):

```xml
<key>NSUserTrackingUsageDescription</key>
<string>This identifier will be used to deliver personalized ads to you.</string>
```

> iOS **aborts the process at launch** if any ATT-backed API is called without `NSUserTrackingUsageDescription`. Customize the string to match your app's tone.

#### Configure Podfile

The plugin transitively depends on **Google-Mobile-Ads-SDK 12.x**, which ships as a static binary. Default `use_frameworks!` (dynamic) is incompatible with static transitive dependencies — switch the linkage to `:static`:

```ruby
target 'Runner' do
  use_frameworks! :linkage => :static    # ← was: use_frameworks!

  flutter_install_all_ios_pods File.dirname(File.realpath(__FILE__))
end
```

Additionally, `NovvyAds.xcframework` only ships `arm64` simulator slices. Exclude `x86_64` from simulator builds:

```ruby
post_install do |installer|
  installer.pods_project.targets.each do |target|
    flutter_additional_ios_build_settings(target)
    target.build_configurations.each do |config|
      config.build_settings['EXCLUDED_ARCHS[sdk=iphonesimulator*]'] = 'x86_64'
    end
  end

  installer.aggregate_targets.each do |aggregate|
    aggregate.user_project.native_targets.each do |target|
      target.build_configurations.each do |config|
        config.build_settings['EXCLUDED_ARCHS[sdk=iphonesimulator*]'] = 'x86_64'
      end
    end
    aggregate.user_project.save
  end
end
```

---

## Usage

### 1. Implement `NovvyAdsCallback`

Create a class that implements the callback interface. All ad outcomes are delivered here — interstitial, rewarded, banner, and feed events all go to a single callback registered at initialization.

```dart
import 'package:novvy_ads/novvy_ads.dart';

class MyAdsCallback implements NovvyAdsCallback {
  // ── Interstitial (required) ───────────────────────────────────────────

  @override
  void onInterstitialDismissed(String adUnitId) {
    // Ad was shown and the user dismissed it. Proceed to next screen.
  }

  @override
  void onInterstitialFailed(String adUnitId, String error) {
    print('Interstitial failed [$adUnitId]: $error');
  }

  // ── Rewarded (required) ───────────────────────────────────────────────

  @override
  void onRewardedDismissed(String adUnitId, NovvyReward? reward) {
    if (reward != null) {
      // User completed the video — grant the reward.
      grantReward(reward.amount, reward.type);
    }
    // reward == null means the user dismissed early; do not grant.
  }

  @override
  void onRewardedFailed(String adUnitId, String error) {
    print('Rewarded failed [$adUnitId]: $error');
  }

  // ── Banner (optional) ─────────────────────────────────────────────────

  @override
  void onBannerImpression(String adUnitId) {}

  @override
  void onBannerFailed(String adUnitId, String error) {}

  @override
  void onBannerClosed(String adUnitId) {}

  // ── Feed (optional) ───────────────────────────────────────────────────

  @override
  void onFeedAdReady(String adUnitId, NovvyFeedAdProvider provider) {
    // Store provider and insert provider.widget into your feed list.
  }

  @override
  void onFeedAdFailed(String adUnitId, String error) {}

  @override
  void onFeedAdPlaybackTick(String adUnitId, int remainingSeconds) {}

  @override
  void onFeedAdImpression(String adUnitId) {}

  @override
  void onFeedAdClicked(String adUnitId) {}
}
```

### 2. Initialize the SDK

Call `NovvyAds.initialize()` in `main()` before `runApp()`:

```dart
import 'package:flutter/widgets.dart';
import 'package:novvy_ads/novvy_ads.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  await NovvyAds.initialize(
    config: NovvyInitConfig(
      appId:    'YOUR_APP_ID',
      endpoint: 'YOUR_ENDPOINT',
      apiKey:   'YOUR_API_KEY',
    ),
    callback: MyAdsCallback(),
  );

  runApp(const MyApp());
}
```

> `initialize()` may be called multiple times — for example, after user login/logout — to replace the callback and trigger a fresh `/init` request. In-flight ads deliver events to the new callback automatically.

### 3. Set User Profile and Content Context (Recommended)

Providing user and content signals improves ad targeting and revenue. The SDK exposes two separate namespaces:

#### User profile

```dart
// Replace the entire profile (null fields clear existing values):
NovvyAds.setUserProfile(NovvyUserProfile(
  userId:      'your-stable-user-id',
  hashedEmail: NovvyAds.hashEmail('user@example.com'),
  isPaidUser:  true,
  age:         1, // age bucket code defined by your host app — see table below
));

// Merge individual fields (null fields preserve existing values):
NovvyAds.updateUserProfile(NovvyUserProfile(isPaidUser: false));
```

| Field | Type | OpenRTB field | Notes |
|-------|------|--------------|-------|
| `userId` | `String?` | `user.user_id` | Your publisher-side stable user ID |
| `hashedEmail` | `String?` | `user.hashed_email` | SHA-256 hash — always use `NovvyAds.hashEmail(rawEmail)` |
| `isPaidUser` | `bool?` | `user.is_paid_user` | `true` if user has an active paid subscription |
| `age` | `int?` | `user.age` | **Age bucket code, not a literal age.** The bucket scheme is defined by your host app — see below |

> **Privacy:** `NovvyAds.hashEmail(email)` trims, lowercases, and SHA-256 hashes the input. Never pass a raw email address.

**Age field — bucket code, not a literal age.** The `age` field is an opaque integer bucket code, **not** the user's age in years. Neither this plugin nor the Novvy SDK ships a predefined enum — the bucket scheme (what each integer means) is **defined by your host app**.

For example, your team might decide:

| Code | Meaning (illustrative only) |
|------|-----------------------------|
| `0`  | Under 15                    |
| `1`  | 15–18                       |
| …    | …                           |

> The values above are **examples only** — use whatever scheme your app team has agreed on. The plugin forwards the integer as-is to the underlying SDK; it does **not** derive the code from a literal age, validate the value, or interpret what the code means.

#### Content context

```dart
// Set whenever the user navigates to new content:
NovvyAds.setContentContext(NovvyContentContext(
  seriesName:    'My Series',
  episodeNumber: 3,
  contentUrl:    'https://example.com/series/ep3',
));
```

| Field | Type | OpenRTB field | Notes |
|-------|------|--------------|-------|
| `seriesName` | `String?` | `app.content.series_name` | Current series / show / channel name |
| `episodeNumber` | `int?` | `app.content.episode_number` | Episode number within the series |
| `contentUrl` | `String?` | `app.content.url` | Canonical URL of the current content |

> Context is read at each ad's `load()` call — set it before triggering ads, and update it whenever the user navigates to new content.

---

### 4. Interstitial Ads

Show a full-screen ad at natural transition points (e.g. between episodes):

```dart
void _onEpisodeEnd() {
  NovvyAds.showInterstitial('YOUR_INTERSTITIAL_AD_UNIT_ID');
  // Result → onInterstitialDismissed or onInterstitialFailed
}
```

The plugin manages the full lifecycle (load → show → dispose) internally. Updating content context before calling `showInterstitial` ensures the bid request reflects the current episode.

---

### 5. Rewarded Ads

Offer users a reward in exchange for watching a video to completion:

```dart
void _onUnlockContent() {
  NovvyAds.showRewarded('YOUR_REWARDED_AD_UNIT_ID');
  // Result → onRewardedDismissed(adUnitId, reward)
  //   reward != null → user completed the video → grant reward
  //   reward == null → user dismissed early → do not grant
}
```

---

### 6. Banner Ads

Banner ads are 260×90 dp native views that embed inside your widget tree. The SDK supports two server-driven display modes — no client-side code changes are needed to switch between them:

| Mode | When the banner shows | Typical use case |
|------|-----------------------|-----------------|
| **Playback Position** | Within a time window (start ms → end ms) configured on the server; the plugin forwards `currentPositionMs` to the native SDK at ~500 ms intervals so it can evaluate the window | Mid-roll or post-roll overlay timed to content |
| **Play / Pause** | When `isPlaying` is `false` (player paused or stopped); hides automatically when `isPlaying` returns to `true` | Non-intrusive ads that only appear during user-initiated pauses |

#### Implement `NovvyPlayerAdapter`

Bridge your video/audio player so the SDK can read its state:

```dart
class MyPlayerAdapter extends NovvyPlayerAdapter {
  final MyVideoPlayer _player;

  MyPlayerAdapter(this._player);

  @override
  int get currentPositionMs => _player.position.inMilliseconds;

  @override
  bool get isPlaying => _player.isPlaying;
}
```

Both properties are polled at ~500 ms intervals and forwarded to the native SDK. Keep the getters lightweight (no I/O or locks).

- `currentPositionMs` — return the raw player position with no caching or rounding.
- `isPlaying` — return `true` only when the player is **actively playing**. Return `false` during pause, buffering/loading, and seek. Both Play/Pause mode and Playback Position mode rely on this to decide when to hide the banner.

#### Display the banner

Place `NovvyAds.banner()` inside a `Stack`. The widget auto-manages loading, server-driven positioning, visibility, and disposal:

```dart
class _MyPageState extends State<MyPage> {
  late final MyPlayerAdapter _playerAdapter;

  @override
  void initState() {
    super.initState();
    _playerAdapter = MyPlayerAdapter(myVideoPlayer);
  }

  @override
  Widget build(BuildContext context) {
    return Stack(
      children: [
        MyVideoContent(),
        NovvyAds.banner(
          adUnitId: 'YOUR_BANNER_AD_UNIT_ID',
          player: _playerAdapter,
        ),
      ],
    );
  }
}
```

- The banner is **260 dp wide × 90 dp tall**, left-aligned.
- Position (top/bottom, vertical offset) is server-driven — no client code changes needed.
- When hidden, the widget is fully transparent and does not intercept touch events.

---

### 7. Feed Ads

Feed ads insert into scrollable lists (e.g. episode lists, article feeds). Two creative formats are supported — the plugin selects the renderer automatically:

- **Image creative** — Native hero image with product info overlay
- **H5 zip creative** — WebView rendering a downloaded HTML/CSS/JS bundle (with optional video). Bundles are cached on disk by `material_url`.

Playback is visibility-driven: you call `setPlaying(true/false)` as the item scrolls in and out of view.

> Feed ads are supported on both **Android and iOS**.

#### Step 1 — Implement feed callbacks

```dart
class MyAdsCallback implements NovvyAdsCallback {
  // ... (interstitial/rewarded/banner callbacks) ...

  NovvyFeedAdProvider? _feedProvider;

  @override
  void onFeedAdReady(String adUnitId, NovvyFeedAdProvider provider) {
    _feedProvider = provider;
    // Insert provider.widget into your feed — see Step 3
  }

  @override
  void onFeedAdFailed(String adUnitId, String error) {
    print('Feed ad failed [$adUnitId]: $error');
  }

  @override
  void onFeedAdPlaybackTick(String adUnitId, int remainingSeconds) {
    // Fires every second while the ad plays.
    // Use to show a custom countdown timer if useDefaultSwipeOverlay is false.
  }
}
```

#### Step 2 — Load the feed ad

```dart
NovvyAds.loadFeedAd(
  adUnitId: 'YOUR_FEED_AD_UNIT_ID',
  durationSeconds: 4,           // seconds the user must wait before swiping past (default: 4, min: 3)
  useDefaultSwipeOverlay: true, // SDK renders built-in "Swipe to skip in Ns" pill
  // episodeNumber: 3,          // optional: override episode for bid request without changing global context
);
// Result → onFeedAdReady or onFeedAdFailed
```

Load the ad ahead of time (e.g. when the page opens) so it is ready before the user scrolls to the insertion point.

> **`useDefaultSwipeOverlay: false`** — When disabled, the SDK does not render any countdown UI. You are responsible for showing a custom overlay. Drive it off `onFeedAdPlaybackTick(adUnitId, remainingSeconds)`, which fires every second with the seconds remaining until the user can swipe past.
>
> **`episodeNumber`** — An optional per-request override for the episode number used in the bid request. When set, it takes precedence over the value previously set via `setContentContext`, without mutating the global context. Useful when loading multiple feed ads for different episodes on the same page.

#### Step 3 — Insert into the feed

```dart
List<Object> _feedItems = [...]; // your feed data
NovvyFeedAdProvider? _feedAdProvider;

@override
void onFeedAdReady(String adUnitId, NovvyFeedAdProvider provider) {
  setState(() {
    _feedAdProvider = provider;
    _feedItems.insert(3, provider); // insert at desired position
  });
}

Widget _buildFeedItem(BuildContext context, int index) {
  final item = _feedItems[index];
  if (item is NovvyFeedAdProvider) {
    return VisibilityDetector(
      key: Key('feed-ad-$index'),
      onVisibilityChanged: (info) {
        item.setPlaying(info.visibleFraction > 0.5);
      },
      child: SizedBox(height: 250, child: item.widget),
    );
  }
  return NormalFeedItem(item: item as MyFeedItem);
}
```

> The [`visibility_detector`](https://pub.dev/packages/visibility_detector) package is recommended for tracking scroll visibility.

#### Step 4 — Dispose when done

```dart
@override
void dispose() {
  _feedAdProvider?.dispose();
  super.dispose();
}
```

Removing the widget from the tree only detaches the native view — you **must** call `dispose()` explicitly to release native resources.

---

## API Reference

### `NovvyAds`

| Method | Description |
|--------|-------------|
| `initialize({config, callback})` | Initialize the SDK. Must complete before showing ads. |
| `setUserProfile(NovvyUserProfile)` | Replace the entire user profile (null fields clear existing values). |
| `updateUserProfile(NovvyUserProfile)` | Merge individual user profile fields (null fields preserve existing values). |
| `setContentContext(NovvyContentContext)` | Replace the entire content context (null fields clear existing values). |
| `showInterstitial(adUnitId)` | Load and show an interstitial. Result via callback. |
| `showRewarded(adUnitId)` | Load and show a rewarded video. Result via callback. |
| `banner({adUnitId, player})` | Return a self-managing banner widget. Place in a `Stack`. |
| `loadFeedAd({adUnitId, durationSeconds, useDefaultSwipeOverlay, episodeNumber})` | Load a feed ad. Result via `onFeedAdReady`. |
| `hashEmail(email)` | SHA-256 hash an email for privacy compliance. |

### `NovvyAdsCallback`

| Method | Required | Description |
|--------|----------|-------------|
| `onInterstitialDismissed(adUnitId)` | Yes | Interstitial was shown and dismissed. |
| `onInterstitialFailed(adUnitId, error)` | Yes | Interstitial failed to load or show. |
| `onRewardedDismissed(adUnitId, reward)` | Yes | Rewarded ad dismissed; `reward` non-null = user earned it. |
| `onRewardedFailed(adUnitId, error)` | Yes | Rewarded ad failed to load or show. |
| `onBannerImpression(adUnitId)` | No | Banner impression recorded. |
| `onBannerFailed(adUnitId, error)` | No | Banner failed to load. |
| `onBannerClosed(adUnitId)` | No | User closed banner via close button. |
| `onFeedAdReady(adUnitId, provider)` | No | Feed ad loaded; insert `provider.widget` into your list. |
| `onFeedAdFailed(adUnitId, error)` | No | Feed ad failed to load. |
| `onFeedAdPlaybackTick(adUnitId, remainingSeconds)` | No | Fires every second during playback. `remainingSeconds` counts down from `durationSeconds` to 0 — the user can swipe past the ad when it reaches 0. Use this to render a custom countdown UI when `useDefaultSwipeOverlay` is `false`. |
| `onFeedAdImpression(adUnitId)` | No | Feed ad impression recorded. |
| `onFeedAdClicked(adUnitId)` | No | Feed ad clicked. |

### `NovvyInitConfig`

| Field | Type | Description |
|-------|------|-------------|
| `appId` | `String` | Your application ID |
| `endpoint` | `String` | Novvy API endpoint URL |
| `apiKey` | `String` | Your API key |

### `NovvyUserProfile`

All fields are optional. Use `setUserProfile` to replace the entire profile; use `updateUserProfile` to merge individual fields.

| Field | Type | `setUserProfile` | `updateUserProfile` | Description |
|-------|------|-----------------|---------------------|-------------|
| `userId` | `String?` | null clears | null preserves | Your publisher-side stable user identifier |
| `hashedEmail` | `String?` | null clears | null preserves | SHA-256 hashed email — always use `NovvyAds.hashEmail(...)` |
| `isPaidUser` | `bool?` | null clears | null preserves | Whether the user has an active paid subscription |
| `age` | `int?` | null clears | null preserves | Age bucket code (defined by host app, not literal age) — see [User profile](#user-profile) |

### `NovvyContentContext`

All fields are optional.

| Field | Type | Description |
|-------|------|-------------|
| `seriesName` | `String?` | Current series / show / channel name |
| `episodeNumber` | `int?` | Episode number within the series |
| `contentUrl` | `String?` | Canonical URL of the current content |

### `NovvyPlayerAdapter` (abstract)

| Property | Type | Description |
|----------|------|-------------|
| `currentPositionMs` | `int` | Current playback position in milliseconds |
| `isPlaying` | `bool` | Whether the player is currently playing (not paused or buffering) |

### `NovvyFeedAdProvider`

| Member | Type | Description |
|--------|------|-------------|
| `widget` | `Widget` | The native ad view — wrap in a `SizedBox` with a fixed height |
| `setPlaying(bool)` | `void` | `true` when visible in viewport, `false` when scrolled out |
| `dispose()` | `void` | Release all native resources. **Must** be called manually. |

### `NovvyReward`

| Field | Type | Description |
|-------|------|-------------|
| `amount` | `int` | Reward amount |
| `type` | `String` | Reward type identifier |

---

## Troubleshooting

### App crashes at launch

**Android — "The Google Mobile Ads SDK was initialized incorrectly":**
- Add `<meta-data android:name="com.google.android.gms.ads.APPLICATION_ID" .../>` inside `<application>` in `AndroidManifest.xml`
- Confirm the value starts with `ca-app-pub-`

**iOS — `__abort_with_payload` / "GADMobileAds ERROR: Initialize with correct APPLICATION_ID":**
- Add `GADApplicationIdentifier` to `ios/Runner/Info.plist`
- Run `flutter clean && flutter run` after editing the plist

**iOS — "app attempted to access privacy-sensitive data without a usage description":**
- Add `NSUserTrackingUsageDescription` to `ios/Runner/Info.plist`
- Run `flutter clean && flutter run`

### Ads not loading

**`onInterstitialFailed` / `onRewardedFailed` fires with "No active Activity" / "No active ViewController":**
- The ad was ready to show but there was no foreground Activity (Android) or ViewController (iOS) at that moment — common during screen rotation or rapid navigation
- Retry `showInterstitial()` / `showRewarded()` once the screen has stabilized

**`onInterstitialFailed` / `onRewardedFailed` fires immediately:**
- Verify `appId`, `endpoint`, and `apiKey` in `NovvyInitConfig`
- Confirm the device has an active internet connection
- Check logcat / Xcode console for detailed error output

**Assertion error — "NovvyAds.initialize() must be called before showInterstitial()":**
- `showInterstitial()` or `showRewarded()` was called before `initialize()` completed
- Ensure `await NovvyAds.initialize(...)` finishes in `main()` before `runApp()`

### Android build errors

**`minSdk` conflict:**
```bash
# Set minSdk 24 in android/app/build.gradle, then:
flutter clean && flutter pub get
```

**Gradle sync fails:**
- Run `./gradlew dependencies` in `android/` to inspect the dependency tree
- Ensure `mavenCentral()` is in your project-level `repositories`

### iOS build errors

**"Unable to find matching slice in '...' for ... x86\_64":**
- Add the `EXCLUDED_ARCHS[sdk=iphonesimulator*] = x86_64` snippet to `ios/Podfile` (see [Configure iOS](#3-configure-ios))
- Run `pod install` from `ios/` and rebuild

**"Module 'novvy\_ads' not found":**
- Usually the same root cause as above — verify `EXCLUDED_ARCHS` is set in the `post_install` hook

**`pod install` fails to download `NovvyAds.xcframework.zip`:**
- Check connectivity to `github.com`
- Manually download the xcframework and place it under the plugin's `ios/` directory; subsequent `pod install` runs reuse it

---

## <a id="ios-notes"></a>iOS Notes

### AdMob mediation

The AdMob mediation adapter is **bundled with this Flutter plugin out of the box** — the adapter source is pulled in automatically by `pod install`, and the plugin transitively depends on `Google-Mobile-Ads-SDK ~> 12.0`. No extra Podfile configuration is required.

If your app uses Google AdMob and you want Novvy as one of its waterfall networks, register `NovvyAdMobMediationAdapter` as a **Custom Event** in the AdMob console for each ad unit. See the [NovvyAds iOS SDK README](https://github.com/NovvyAI/novvy-ads-cocoapods) for the full mediation setup walkthrough.

This is independent of `NovvyAds.showInterstitial()` / `showRewarded()` calls from Dart — those always use the Novvy direct path.

### Simulator architecture

The NovvyAds xcframework ships `ios-arm64` and `ios-arm64-simulator` slices only. Apple Silicon Macs run the simulator as `arm64` and work fine. Intel Macs and Intel-based CI runners cannot run the iOS simulator until the upstream SDK ships an `x86_64` slice. **Real-device builds work from any Mac.**

---

## Requirements

| | Minimum |
|-|---------|
| Dart | 3.0.0+ |
| Flutter | 3.10.0+ |
| Android | API 24 (Android 7.0) |
| iOS | 13.0 (Xcode 14+, Swift 5.0+) |

## Additional Resources

- [Example App](./example)
- [Novvy AI](https://novvy.ai/)

## Support

Email: founders@novvy.ai

## License

MIT License — Copyright (c) 2026 Novvy AI
