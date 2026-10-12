# Flutter App UI Explorer: plan

Status: **planning, no code yet** · Written 2026-10-12 · Owner: Daniel Fransix

## 1. Goal

Build the Flutter equivalent of [Web App UI Explorer](https://github.com/danielfransix/web-app-ui-explorer): point it at a Flutter app and it
**discovers every screen and state on its own**, captures a screenshot of each, and shows them as a gallery, a flow canvas and a
source map, with an optional read-only snapshot for an internal URL.

What "done" looks like on a real Flutter app:

- Finds every top-level screen and most reachable states **without hand-written scenarios**.
  Reference point from the web tool, measured on a 7-step wizard: all 13 screens and 58-82% of 50 hand-listed states, depending on machine load.
- Every state has a **recipe** (the steps from a clean launch) that replays reliably.
- Screenshots at phone and tablet sizes, light and dark.
- Each state links to the Dart file that builds it.
- The output opens in the same viewer experience as the web tool.

Non-goals for now: pixel-diff regression (golden tests already do this), end-to-end test authoring, and driving native OS dialogs.

## 2. The reference: what the web tool does

1. Opens the app at a start path in a fresh page.
2. Lists what a person could act on now (buttons, tabs, switches, dropdowns). Anything under an open dialog is excluded.
3. For each one: reload, replay the recipe so far, perform the step, look at the result.
4. Fingerprints each result (headings, dialogs, selections, visible text) and keeps only new ones.
5. Fills forms with believable values; blocks requests that change data.
6. Writes `scenarios.json`, then maps each state to the source file that holds its text.

Every recipe starts from a clean load, so any state can be reproduced. The viewer only reads the resulting data, which is what makes a second "producer" for Flutter realistic.

## 3. Landscape (researched 2026-10-12)

**No tool found that does automatic discovery plus screenshots plus a gallery and flow canvas for Flutter.**

| Tool | What it does | Gap for us |
|---|---|---|
| [Dangi Doctor](https://pub.dev/documentation/dangi_doctor/0.3.0/) | Crawls a running app over the Dart VM service by tapping navigation; HTML health report; generated integration tests | Closest. Android only per its page, v0.3.0, screenshots not mentioned, no independent reviews found |
| [Firebase Test Lab Robo](https://firebase.google.com/docs/test-lab/ios/robo-ux-test) | Crawls an app: screenshots, video, screens-and-actions graph | [Docs say Robo does not natively support Flutter](https://firebase.google.com/docs/test-lab/flutter/integration-testing-with-flutter?hl=de); needs Robo scripts |
| [Widgetbook](https://docs.widgetbook.io/why-widgetbook) | "Storybook for Flutter": hand-written use cases, knobs, device/theme/locale switching | Manual catalog by design. Cloud review product's status unverified (docs page 404'd) |
| [Flutter Widget Previewer](https://docs.flutter.dev/tools/widget-previewer) | Official previewer for `@Preview` widgets; docs list it as stable from Flutter 3.47 | Only annotated widgets; runs on Flutter Web, so no `dart:io` or native plugins |
| [Alchemist](https://verygood.ventures/blog/alchemist-golden-tests-tutorial.md) (golden_toolkit is [discontinued](https://pub.dev/packages/golden_toolkit)) | Golden tests across device sizes | One hand-written test per screen |
| [auto_screenshot](https://github.com/spielgestalt/auto_screenshot), [screenshots](https://github.com/mmcc007/screenshots), [screenshooter](https://pub.dev/packages/screenshooter) | Screenshots from a list of pages or test steps | You list every screen |
| [cartographer](https://pub.dev/documentation/cartographer/latest/), [buoy_routes](https://pub.dev/packages/buoy_routes/example) | Static navigation graph; live go_router inspector | cartographer does not detect go_router yet |

Patrol, Maestro and Appium are test-automation tools you script, not explorers. The comparison articles found are mostly vendor blogs.

## 4. Approaches

| | A. Flutter Web + semantics, reuse the web explorer | B. In-process crawler on `integration_test` / widget tests | C. External crawler over the Dart VM service |
|---|---|---|---|
| Idea | Build the app for web, switch on Flutter's [accessibility tree](https://docs.flutter.dev/ui/accessibility/web-accessibility) so it appears in the DOM, crawl with the existing Playwright crawler | Same algorithm, written in Dart, running inside Flutter's test harness; reads the semantics tree | Attach to a running app on a device or emulator (what Dangi Doctor does) |
| Effort | Lowest | Medium | Medium-high |
| Fidelity | Web layout, not mobile; plugins differ | Real app code and layout; plugins stubbed | Highest: real device and plugins |
| Speed / CI | Fast | Fast, headless | Slow, flaky-prone |
| Source mapping | Text matching only | Widget creation locations (debug builds) | Widget creation locations |
| Risk | Feasibility is **my inference**, not confirmed | Fakes and stubs needed; semantics gaps | Device management; Android-first |

**Decision (proposed):** spike **A** to prove the viewer and data flow end to end, build **B** as the real product,
revisit **C** only if device-level fidelity becomes a requirement.

## 5. Architecture

Split the system into a **producer** (finds and records states) and a **viewer** (shows them).

- **Shared data format.** Scenario records: `id`, `title`, `group`, `kind` (screen, overlay or variant), `parent`, `trigger`,
  `actions` (the recipe), `source` (file), and a thumbnail reference. The web tool's static snapshot (`api/state.json`) is the starting point.
- **Viewer.** Reuse the web explorer's page. Decide in Phase 0 whether to extract it into a shared package or copy its built client into
  the Flutter tool's output.
- **Flutter producer** (a Dart dev dependency, published to pub.dev later):
  - crawler core: candidates, replay, fingerprint, budget;
  - semantics adapter: finds tappable, editable and scrollable things;
  - screenshot adapter: renders the widget tree to an image at each device size and theme;
  - source mapper: widget creation locations via the inspector's [creation tracking](https://api.flutter.dev/flutter/widgets/WidgetInspectorService/isWidgetCreationTracked.html);
  - CLI: `dart run flutter_app_ui_explorer:crawl`, plus a config for deny-list, devices and the fakes entrypoint.
- **Entrypoint with fakes.** The app under exploration is started from a small `explorer_main.dart` the team owns, which swaps in
  fake repositories, auth and network so screens render without a backend.

### Crawl loop (Flutter)

1. Launch the app from `explorer_main.dart` in a fresh test harness at a chosen device size.
2. Read the semantics tree; list actionable nodes (taps, text fields, switches, scrollables). Exclude anything under a modal barrier.
3. For each: relaunch, replay the recipe so far, perform the step, wait until settled (with a timeout for endless animations).
4. Fingerprint: current route or page, dialog and sheet labels, selected states, visible semantics labels.
5. Keep new fingerprints; screenshot; record source location; enqueue its actions.
6. Fill text fields with believable values; make a first choice in empty pickers; try back and close with unsaved changes.
7. Cap states, depth and time; write `scenarios.json` and the images.

## 6. Phases

Estimates are **guesses** for one person and depend on the target app.

**Phase 0: inputs and decisions (before any code)**
- Target app and repo, platforms (iOS, Android, web), router (go_router?), state management, how screens get data.
- Decide viewer reuse (extract vs copy) and the scenario format version.
- Exit: written answers to the open questions in section 8.

**Phase 1: spike (1-2 days)**
- Build the target app for web, enable semantics, run the existing crawler, open the result in the viewer.
- In parallel (half a day): run Dangi Doctor on the same app as a baseline.
- Exit: go or no-go. Go if the crawl finds the main screens and the viewer shows them. Note what semantics gaps exist.

**Phase 2: crawler MVP in Dart (about 2-3 weeks)**
- Harness, semantics reader, tap and replay, fingerprint, settle logic, screenshots at one device size, JSON output.
- Exit: on the target app, finds all top-level screens and a meaningful share of states; recipes replay.

**Phase 3: fidelity (about 2 weeks)**
- Text entry, scrolling to reveal, back and dismiss, long-press, swipe; device sizes, text scale, dark mode;
  creation-location source mapping; deny-list; failure reporting.

**Phase 4: viewer integration and deploy (about 1 week)**
- Viewer reads Flutter output; read-only snapshot for an internal URL; thumbnail freshness tied to a source hash.

**Phase 5: package and docs (about 1 week)**
- Publish to pub.dev, README, example app, CI.

## 7. Considerations and risks

| Topic | Risk | Mitigation |
|---|---|---|
| Data and network | Screens need login or API data | `explorer_main.dart` with injected fakes |
| Plugins, platform channels | Missing in headless tests | Stub them; document a standard set |
| Animations | Endless animations stop "settled" from ever firing | Time-boxed settle; ignore known loops |
| Semantics gaps | Custom-painted widgets without semantics are invisible | Report coverage; add a hint API |
| Gestures | Swipe, long-press and scroll-to-reveal are harder than taps | Phase 3; start with taps and scrolls |
| Fonts | Headless tests render placeholder boxes | Load the app's real fonts in the harness |
| Source mapping | Creation tracking works in debug builds only | Fine for a dev tool; fall back to text match |
| Native UI | `integration_test` [cannot drive native dialogs](https://docs.flutter.dev/cookbook/testing/integration/introduction) | Out of scope; stub permissions |
| State explosion | Device sizes x themes x states multiply | Caps; sample devices; budgets |
| Determinism | Timers, clocks and random data change fingerprints | Fake clock and seeded data |
| Writes | Crawl must not mutate real data | Fakes only; never point at production |

## 8. Open questions

1. Which Flutter app first, and where is the repo?
2. Platforms that matter: iOS, Android, web, or all?
3. Router (go_router, auto_route, Navigator 1.0) and state management (Riverpod, Bloc, Provider)?
4. Can the app run with fakes, or does it hard-depend on a backend?
5. Is a viewer shared with the web tool a requirement, or is a Flutter-only viewer acceptable?
6. Where should the output live: local only, or an internal URL like the web tool's snapshot?
7. Build vs adopt: is a curated Widgetbook catalog enough for part of the need?

## 9. What is verified and what is not

- **Verified from sources:** the tools in section 3 and what their pages say; Widget Previewer behaviour and limits; Flutter's opt-in web accessibility tree; the creation-location tracking API.
- **My inference, not confirmed:** that switching on semantics lets the existing Playwright crawler find Flutter controls (approach A); that approach B's algorithm ports cleanly; effort estimates.
- **Not verified:** Dangi Doctor's real-world quality and screenshots; Widgetbook Cloud's availability; the Patrol and Maestro stability claims (vendor blogs).

## 10. Next step

Answer section 8, then run Phase 1.
