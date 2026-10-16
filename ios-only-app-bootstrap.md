# Bootstrap a new iOS-only app

A repeatable starting point for a **native iPhone app** in the `hoangbkit` portfolio. This guide covers the first working simulator build, repo structure, AppFoundation integration choices, correctness, GitHub Actions, and release readiness. Use it for a **new app**; do not copy an older app wholesale or inherit its features, paywall, permissions or models.

**Default stack:** SwiftUI + Swift 6 / strict concurrency, iOS 26+, Xcode 26+, XcodeGen as project source of truth, Swift Testing or XCTest, optional AppFoundation for shared infrastructure, GitHub Actions **manual-only**. Target **iPhone only**: not iPad, Mac Catalyst, or an iOS app advertised as Mac-compatible. Adjust the minimum supported iOS version only after confirming every dependency supports it.

For CI and build-release details, use the companion [GitHub Actions workflow guide](github-actions-workflows.md). This document is about **bootstrapping the app**, not duplicating that guide.

## 0. Decisions to make before writing code

Record these in the app's README or a short plan:

| Decision | Recommended default | Notes |
| --- | --- | --- |
| App/repo name | One stable PascalCase identifier (`ExampleApp`) | Repo, module, SwiftUI `@main` type, scheme and generated project should match unless a deliberate branding distinction exists |
| Bundle ID | `com.hoangbkit.exampleapp` | Make unique; reserve corresponding App ID in Apple Developer when preparing a device/distribution build |
| Platform | iPhone, iOS 26+ | `TARGETED_DEVICE_FAMILY: "1"` and no Mac Catalyst / Designed for iPhone on Mac |
| Language | English first, localization-ready | Use localized SwiftUI strings and String Catalogs; do not hardcode English into persistence identifiers |
| Deployment | Debug simulator first | No certificates/secrets needed for the first build |
| Data | Local first, minimal persistence | Select SwiftData/file storage only after defining data invariants and migrations |
| Monetization | **Undecided until explicitly specified** | Do not invent Pro gates, StoreKit product IDs or subscription plans |
| Analytics | **Off by default** | No client is required in a bare skeleton; if wired, `enabled: false` until deliberately enabled |
| Shared foundation | AppFoundation when needed | Pin a verified tag; reuse shared commerce/theme/other infrastructure instead of rewriting it |
| Repository workflow | `master` + optional `develop` | Work on feature branches via PR; CI and builds stay manual-only |

Write down the app's **single core user journey** (for example, Library → Detail → Editor) and the three main acceptance checks for that journey. Do not begin by building settings, a paywall and a dashboard before the primary action works.

## 1. Create the repo and files

Create an empty GitHub repository named `ExampleApp` (typically private during initial development), clone it, and start from a clean directory. Add only one iOS app target and one unit-test target initially.

Suggested layout:

~~~text
ExampleApp/
├── .github/
│   └── workflows/                # Add manual workflows after first local build
├── .gitignore
├── AGENTS.md                     # Local repo/agent conventions (optional)
├── AppStore/                     # App Store copy/screenshots later
├── Config/
│   ├── Debug.xcconfig
│   └── Release.xcconfig
├── ExampleApp/
│   ├── App/
│   │   ├── ExampleAppApp.swift
│   │   └── RootView.swift
│   ├── Domain/                   # Models, identities, rules
│   ├── Features/                 # Product-specific screens and state
│   ├── Services/                 # File/network/device integrations
│   ├── Persistence/              # Only when required by the product
│   └── Resources/
│       └── Assets.xcassets/
│           └── AppIcon.appiconset/
├── ExampleAppTests/
│   └── ExampleAppTests.swift
├── README.md
└── project.yml
~~~

Avoid creating empty layers with placeholder protocols or an elaborate architecture up front; add folders as real code arrives. Keep features, screens, navigation, branded UI and product models **inside the app**. Add `.gitignore` entries for `.DS_Store`, `.build/`, `DerivedData/`, `*.xcresult`, `*.xcarchive`, secret xcconfigs and any heavyweight staged assets. Decide explicitly whether a generated `.xcodeproj` is tracked; if the XcodeGen spec is authoritative, prefer **not** tracking it, but follow any repo-specific policy before changing an existing project.

## 2. Define the XcodeGen project

Install and verify tools locally:

~~~bash
xcodebuild -version
swift --version
brew install xcodegen
xcodegen --version
~~~

**Starter `project.yml`:** replace `ExampleApp`, bundle ID, deployment target and team. This minimal project deliberately has **no** commerce, analytics, external package or extra extension target.

~~~yaml
name: ExampleApp

options:
  minimumXcodeGenVersion: 2.46.0
  deploymentTarget:
    iOS: "26.0"
  createIntermediateGroups: true
  developmentLanguage: en
  defaultConfig: Debug

configs:
  Debug: debug
  Release: release

settings:
  base:
    SWIFT_VERSION: 6.0
    SWIFT_STRICT_CONCURRENCY: complete
    SWIFT_DEFAULT_ACTOR_ISOLATION: MainActor
    CODE_SIGN_STYLE: Automatic
    DEVELOPMENT_TEAM: YOUR_TEAM_ID
    TARGETED_DEVICE_FAMILY: "1"
    SUPPORTS_MACCATALYST: NO
    SUPPORTS_MAC_DESIGNED_FOR_IPHONE_IPAD: NO
    ENABLE_PREVIEWS: YES
    SWIFT_EMIT_LOC_STRINGS: YES

targets:
  ExampleApp:
    type: application
    platform: iOS
    deploymentTarget: "26.0"
    sources:
      - path: ExampleApp
    configFiles:
      Debug: Config/Debug.xcconfig
      Release: Config/Release.xcconfig
    settings:
      base:
        PRODUCT_BUNDLE_IDENTIFIER: com.hoangbkit.exampleapp
        PRODUCT_NAME: ExampleApp
        PRODUCT_MODULE_NAME: ExampleApp
        MARKETING_VERSION: "1.0"
        CURRENT_PROJECT_VERSION: "1"
        GENERATE_INFOPLIST_FILE: YES
        INFOPLIST_KEY_CFBundleDisplayName: ExampleApp
        INFOPLIST_KEY_UILaunchScreen_Generation: YES
        ASSETCATALOG_COMPILER_APPICON_NAME: AppIcon

  ExampleAppTests:
    type: bundle.unit-test
    platform: iOS
    deploymentTarget: "26.0"
    sources:
      - path: ExampleAppTests
    dependencies:
      - target: ExampleApp
    settings:
      base:
        GENERATE_INFOPLIST_FILE: YES
        PRODUCT_BUNDLE_IDENTIFIER: com.hoangbkit.exampleapp.tests

schemes:
  ExampleApp:
    build:
      targets:
        ExampleApp: all
    run:
      config: Debug
    test:
      config: Debug
      targets:
        - ExampleAppTests
    archive:
      config: Release
    management:
      shared: true
~~~

Start with minimal configuration files:

`Config/Debug.xcconfig`
~~~xcconfig
SWIFT_ACTIVE_COMPILATION_CONDITIONS = DEBUG
~~~

`Config/Release.xcconfig`
~~~xcconfig
// Release-specific overrides only, if needed.
~~~

Never place API tokens, signing private keys or real backend secrets into tracked xcconfigs, project YAML, Info.plist, `UserDefaults` or an app binary. If configuration is needed, check in `*.example` templates and ignore local overrides; remember that a "client app key" shipped in the binary is **not** a secret and cannot protect provider credentials. Keep provider/admin credentials server-side.

If adding a static `Info.plist`, use exactly one consistent approach — an XcodeGen-generated Info.plist plus `INFOPLIST_KEY_*` keys, or a managed physical plist — and check that generated build settings point to the intended one. Add usage-description keys **only** for capabilities the app actually uses, with clear descriptions.

## 3. Add a clean SwiftUI entry and a real root screen

`ExampleApp/App/ExampleAppApp.swift`:

~~~swift
import SwiftUI

@main
@MainActor
struct ExampleAppApp: App {
    var body: some Scene {
        WindowGroup {
            RootView()
        }
    }
}
~~~

`ExampleApp/App/RootView.swift`:

~~~swift
import SwiftUI

struct RootView: View {
    var body: some View {
        NavigationStack {
            ContentUnavailableView(
                "Ready to Build",
                systemImage: "square.stack",
                description: Text("Replace this screen with the app's primary flow.")
            )
            .navigationTitle("ExampleApp")
        }
    }
}
~~~

`ExampleAppTests/ExampleAppTests.swift`:

~~~swift
import XCTest
@testable import ExampleApp

final class ExampleAppTests: XCTestCase {
    func testAppBootstrapSmoke() {
        XCTAssertEqual("ExampleApp".isEmpty, false)
    }
}
~~~

The test above is a **temporary compilation smoke test**, not a claim of product correctness. Replace it immediately with a meaningful model, persistence or feature test once the first vertical slice exists.

For iOS UI, prefer native `NavigationStack`, toolbar items, search (`.searchable`), sheets, empty states, content scrolling beneath the navigation bar and SwiftUI materials before custom bars. Keep search under the native navigation title when the screen needs search. Avoid permanently opaque faux navigation headers, large sticky custom hero sections and redundant action buttons. Design the key screen around **one primary action** and a clear visual hierarchy.

## 4. Generate, build and test locally

From the repo root:

~~~bash
xcodegen generate --spec project.yml
test -f ExampleApp.xcodeproj/project.pbxproj
test -f ExampleApp.xcodeproj/xcshareddata/xcschemes/ExampleApp.xcscheme

xcodebuild -project ExampleApp.xcodeproj -scheme ExampleApp \
  -configuration Debug \
  -destination 'generic/platform=iOS Simulator' \
  -derivedDataPath "$PWD/.derivedData" \
  CODE_SIGNING_ALLOWED=NO build
~~~

For tests, first choose an actual available iPhone simulator (`xcrun simctl list devices available`), then run:

~~~bash
xcodebuild -project ExampleApp.xcodeproj -scheme ExampleApp \
  -configuration Debug \
  -destination 'platform=iOS Simulator,id=REPLACE_WITH_AVAILABLE_UDID' \
  -derivedDataPath "$PWD/.derivedData" \
  CODE_SIGNING_ALLOWED=NO test
~~~

The **generic Simulator destination works for build but not for test**. Verify a clean launch, icon, native navigation and both light/dark appearances in the simulator. On a physical iPhone later, choose the correct Apple team and verify signing; the CI unsigned build does not replace local signing.

## 5. Adopt AppFoundation selectively

AppFoundation is the shared iOS foundation for commerce, entitlement evaluation, themes, settings, onboarding helpers, analytics, exports, screenshots, etc. It is **not** the place for app-specific screens, data models or feature logic.

- Add `https://github.com/hoangbkit/AppFoundation.git` to `project.yml` **only when a shared capability is needed**. Pin an **existing verified release tag** with `exactVersion`; don't copy a stale version number from another repo, a README or an unverified latest label. Declare the actual needed products under the app target's `dependencies` and regenerate the Xcode project.
- **Commerce:** decide product IDs, plan types, free boundaries, Pro features and renewal/restore UX before introducing `PurchaseManager`, `PurchaseConfiguration`, `.managesPurchases` or paywall gates. Do not fabricate a purchase policy at bootstrap. When enabled, use verified `hasPro` from AppFoundation; never persist an independent "Pro" Boolean as authorization.
- **Themes:** define app-owned branded light/dark bases and IDs, with free fallbacks. If using `ThemeManager`, create it once at app scope and attach it to the SwiftUI environment. When theme access depends on Pro, follow AppFoundation's current purchase-aware `.appFoundationTheme(themeManager, purchaseManager: purchases)` contract. Do not add competing manual entitlement synchronization mechanisms without a reason.
- **Analytics:** default to **not constructing a client**. If an app intentionally prepares a dormant integration, `AppAnalyticsConfiguration(..., enabled: false)` is fully inert according to AppFoundation's analytics contract. Do not enable without an explicit privacy/product choice; never include user text, filenames, content, receipts or sensitive metadata in events.
- **Settings / onboarding / export / backup / screenshots:** adopt the narrow shared component **only** when the product flow requires it; keep custom navigation and branded views app-owned.

Reference the source-of-truth package docs at [AppFoundation](https://github.com/hoangbkit/AppFoundation), especially [Adoption Guide](https://github.com/hoangbkit/AppFoundation/blob/master/Documentation/AdoptionGuide.md) and [Analytics](https://github.com/hoangbkit/AppFoundation/blob/master/Documentation/Analytics.md). Verify APIs against the pinned tag before copying example code: main-branch documentation and package tags can drift.

## 6. Product skeleton: build a vertical slice, not scaffolding theater

Implement the first **complete** user journey before increasing surface area:

1. A clear first launch and empty state that leads to the primary action.
2. A primary feature with working success, loading, cancellation and error states.
3. Durable storage **only if the feature needs it**, with stable IDs, atomic saves and understandable recovery on corrupted/missing data.
4. Navigation/back/dismiss behavior: avoid zombie sheets, duplicate tasks, invisible player overlays or stale state after reopening the app.
5. Local-first behavior where sensible; explicitly define what happens offline and when the app backgrounds/foregrounds.
6. A meaningful test for one critical domain rule and one feature failure/retry. Add regression tests for files, complex imports, caching or payments as those features appear.

Prefer a small `Domain` layer for invariants, a lightweight persistence boundary, and feature-owned state. Do **not** prematurely build a generic DI container, app-wide event bus, custom router, deeply nested coordinators, or a paid tier solely to make the template look complete.

**Capability rule:** request camera, photo-library, microphone, speech, background-audio, notifications, document access, App Groups, Keychain Sharing or network entitlements only if the chosen feature explicitly requires them. Describe usage in permission strings. App Groups and widgets need matching identifiers and provisioning; don't scaffold them by default.

## 7. Git and manual GitHub Actions

- Create `master` and optionally `develop` in the repository; do development in branches and PRs. Keep the new repo's default branch and scheme names consistent.
- After the first local build succeeds, introduce `fast-ci.yml`, `full-ci.yml` and `release.yml` using [the workflow standard](github-actions-workflows.md). All are **`workflow_dispatch` only**: no automatic `push`, `pull_request`, scheduled or tag triggers.
- **Fast CI:** build the iOS Simulator app and run quick tests on one Apple Silicon runner, no heavy production assets unless strictly necessary.
- **Full CI:** run the broader meaningful iOS Simulator tests, integration/correctness checks and archive-resource validation; an iOS-only app needs **no macOS target or Intel matrix**.
- **Release:** manual iOS **device** `.xcarchive` with `CODE_SIGNING_ALLOWED=NO`, plus a ZIP of the **actual generated `.xcodeproj`** containing `project.pbxproj` and shared scheme. Publish as a GitHub prerelease tagged `mycli-build-<run>-<attempt>`; this is for local `mycli` signing/export/deploy, **not App Store submission**. Verify bundle ID, offline assets, ZIP integrity, per-file size, expected artifact count and selected ref SHA.
- If the repository uses XcodeGen, stage any binary resource directories **before generation** and package the project built in the same run. Do not ship just `project.yml` or an `.app` in place of a full archive.
- Keep workflow access least-privileged (`contents: read` for CI; `contents: write` for prerelease publishing), and never put certificates or signing secrets in the unsigned build job.

**Important:** the manual dispatch UI normally requires the workflow file to exist on the **default branch**; after merging workflows there, choose `develop` or another branch when launching them.

## 8. App Store and device release readiness

Before submitting version 1.0:

- Reserve the final bundle ID and configure the app in Apple Developer / App Store Connect; choose capabilities, signing and distribution settings based on actual product requirements.
- Finalize display name, icon, category, version/build number, deployment floor, supported devices, orientation, privacy policy, support URL and customer-facing description. Keep `AppStore/` assets and metadata ready for the repo's `mycli` workflow.
- Verify correct app icon in both light/dark modes where relevant; test Dynamic Type, VoiceOver accessibility labels, small-screen layouts, loading and empty states.
- Test cold start, repeat start, reinstall, app-switch/background/foreground, interrupted tasks, no network, denied permissions, persistence recovery, and real-device behavior.
- Confirm the app's data practices against App Store privacy responses, privacy manifests and any required permission/reason APIs; do not copy another app's privacy declaration.
- If StoreKit is deliberately added: test purchase, restore, expiry, network loss, purchase cancellation and existing user data availability after downgrade before locking features behind Pro.
- Build and archive with the actual bundled production resources. Review crash/log output, archive identity, entitlement/capability correctness and required content **inside the archive**, not only in source directories.
- Follow the selected app's local signing and TestFlight/App Store procedure; an unsigned `mycli-build-*` prerelease is **not** itself store-ready.

## Completion criteria

- [ ] Unique name, bundle ID, platform and one core flow explicitly chosen.
- [ ] Only one iOS application target (plus initial unit tests), iPhone-only, no unwanted Mac/iPad configuration.
- [ ] `project.yml` regenerates a shared scheme and the Debug Simulator build from a clean checkout.
- [ ] SwiftUI root launches, uses native navigation and presents a genuine primary action/empty state.
- [ ] A real domain or integration test replaces the placeholder smoke assertion.
- [ ] App-specific domain and UI stay in the app; AppFoundation added only as needed and pinned to a verified tag.
- [ ] No invented Pro restrictions, purchase identifiers, analytics collection, permissions, model downloads or backend secrets.
- [ ] Local data survives relaunch and handles failure if the app stores durable user work.
- [ ] iPhone simulator and a real iPhone both work; dark mode, small screens and interrupted lifecycle tested.
- [ ] Fast CI, Full CI and Release use **manual triggers only**, matching the existing workflow guide.
- [ ] Release produces unsigned iOS `.xcarchive.zip` **and** a separate generated `.xcodeproj.zip` for `mycli`.
- [ ] App Store metadata, privacy, entitlements, signing, screenshots and recovery checks completed before shipping.

## Reference apps

These are examples of **current patterns**, not templates to copy verbatim:

- [ReadAloud](https://github.com/hoangbkit/ReadAloud) (develop): iPhone-only iOS 26+ XcodeGen app, app-scoped SwiftUI state, AppFoundation purchases/themes, offline resources and manual release with generated project ZIP.
- [MenuLens](https://github.com/hoangbkit/MenuLens): iPhone-only app with configured Debug/Release targets, persistence/AI boundaries, App Store metadata and AppFoundation commerce.
- [Milesto](https://github.com/hoangbkit/Milesto): additional optional widget and App Group example; **do not** create these for a simple new app.
- [AppFoundation adoption](https://github.com/hoangbkit/AppFoundation/blob/master/Documentation/AdoptionGuide.md): shared infrastructure contract.

**Guiding principle:** bootstrap the smallest correct, beautiful, working iPhone app; attach shared infrastructure only at well-defined seams, then expand after the core loop is stable.
