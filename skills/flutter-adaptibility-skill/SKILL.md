---
name: flutter-adaptability
description: >
  Use this skill when working on any Flutter project that needs to run well
  across multiple form factors — phones, tablets, foldables, desktops, TV, or
  web. Triggers include requests to "make the app adaptive", "support tablet
  layout", "fix desktop UI", "handle foldables", "add TV support", or any task
  that touches layout, navigation, input handling, or screen-size responsiveness
  in a Flutter codebase.
---

# Flutter Adaptability Skill

This skill turns the AI into a **code-aware adaptability auditor and implementer**.
It does not prescribe a fixed implementation. Instead it reads the actual project,
diagnoses what is missing or wrong, and produces targeted, minimal changes that fit
the existing code style and architecture.

---

## Use Case Scenarios

Depending on the user's request, adapt your workflow to one of these three primary scenarios:

### 1. New Project Creation
- **Action**: Establish the adaptability foundations from day one.
- **Guidelines**: Set up a centralized breakpoint utility or standard `LayoutBuilder` pattern early. Configure `MaterialScrollBehavior` in `main.dart` to support drag devices for desktop/web. Ensure the initial architecture inherently separates the list view and detail view to prepare for multi-pane layouts.

### 2. Fixing Existing Code
- **Action**: Audit and surgically correct non-adaptive layouts.
- **Guidelines**: Search for anti-patterns (e.g., `Platform.isAndroid` for UI decisions, `SystemChrome.setPreferredOrientations`, fixed width columns spanning large screens). Remove them and implement native replacements like `LayoutBuilder` and `NavigationRail`. Do not rewrite the entire app; only patch the broken layout constraints.

### 3. Adding a New Feature
- **Action**: Ensure the new component or screen meets all adaptability standards before completing the task.
- **Guidelines**: Test the new widget across the standard breakpoints (Compact, Medium, Expanded). Verify that it can be navigated via keyboard (Tab/Enter) and doesn't obscure the hinge bounds if placed on a foldable device. If it introduces a new navigation route, ensure it behaves properly within a multi-pane layout if applicable.

---

## Step 1 — Read the project before doing anything

Before writing a single line of code:

1. List the project tree (`lib/`, `android/`, `ios/`, `pubspec.yaml`).
2. Identify the existing navigation pattern, state management approach, and any
   packages already in use.
3. Note the current breakpoint strategy (or absence of one).
4. Check `AndroidManifest.xml` for TV / large-screen declarations.
5. Check `main.dart` for orientation locks, scroll behaviour, and system UI setup.

Only after this scan should any changes be planned.

---

## Step 2 — Diagnose against these core principles

Evaluate the project against these platform-agnostic principles. Do **not** apply
every principle blindly — only flag what is actually absent or broken.

### Layout
- Layout decisions must respond to **available width**, not the device type or OS.
  Use `LayoutBuilder` inside widgets; use `MediaQuery.sizeOf(context)` for
  top-level decisions. Never use `Platform.isAndroid` / `Platform.isIOS` to
  drive layout.
- The Material 3 breakpoint ladder is a good default reference:
  compact < 600 dp · medium 600–839 dp · expanded 840–1199 dp · large ≥ 1200 dp.
  Adjust these thresholds if the existing codebase already defines its own and
  they are reasonable.
- On wide screens, content must not be a stretched single column. Introduce
  multi-column layouts, two-pane (list-detail) splits, or wider content regions
  where it makes sense for the screen.

### Navigation
- Bottom `NavigationBar` suits compact screens; `NavigationRail` suits medium and
  above. The choice between icon-only and extended rail is a detail — match what
  looks right at the available width.
- Navigation must never be hidden behind a hamburger on screens ≥ 600 dp unless
  the project deliberately targets a narrow content experience (e.g. a reading app).
- Orientation must never be locked unless there is a documented, product-level
  reason.

### Input
- Interactive elements must be reachable and activatable without touch: keyboard
  Tab / arrow navigation, Enter / Space activation, and Escape for dismissal.
- Desktop and web builds need hover feedback (`MouseRegion`) and correct cursor
  shapes on interactive elements.
- Right-click context menus should exist on content items in desktop / web builds.
- Stylus events (`PointerDeviceKind.stylus`) should be handled distinctly from
  touch where the app involves drawing, annotation, or precision input.
- `MaterialScrollBehavior` must include mouse and trackpad drag devices for
  desktop and web targets.

### Foldable devices
- If the app targets Android and the existing code has no hinge/fold awareness,
  check whether any screen is likely to break across a fold (e.g. a horizontally
  centred card that would straddle a vertical hinge).
- Use `MediaQuery.displayFeaturesOf(context)` to detect hinges. Never place
  interactive UI over hinge bounds.

### Android TV
- TV requires a `LEANBACK_LAUNCHER` intent-filter and
  `android.hardware.touchscreen android:required="false"` in the manifest.
- Detect TV / D-pad mode via `MediaQuery.navigationModeOf(context) ==
  NavigationMode.directional` — not via `Platform` checks.
- Every tap action needs a keyboard / D-pad equivalent. Focus must be explicit
  and visible. Apply overscan-safe padding (~5 % per edge).

### State restoration
- Screen-level state (selected item, scroll position, search query) should
  survive rotation and foldable posture changes. `RestorationMixin` is the
  SDK-native way; accept equivalent approaches (e.g. a state-management solution
  that persists state across config changes).

---

## Step 3 — Produce targeted changes

After diagnosis:

- **List only the actual gaps found** in the project. Do not mention principles
  that are already satisfied.
- For each gap, produce the **smallest correct change** that fixes it — a new
  widget, a modified build method, a manifest addition, etc.
- Match the code style, naming conventions, and state management pattern already
  present in the project. Do not introduce new packages unless nothing in the SDK
  covers the need.
- If a gap is significant but outside the current task scope, call it out as a
  recommendation without implementing it.

---

## Constraints and anti-patterns to enforce

| Anti-pattern | Correct approach |
|---|---|
| `Platform.isAndroid` / `isIOS` for layout | `LayoutBuilder` constraints |
| Hardcoded width thresholds scattered across files | A single breakpoint utility the rest of the code imports |
| `SystemChrome.setPreferredOrientations` locking to portrait/landscape | Remove or unlock |
| `dart:html window.innerWidth` in web code | `LayoutBuilder` / `MediaQuery.sizeOf` |
| Touch-only `onTap` with no keyboard equivalent | Pair with `onKeyEvent` for Enter/Select |
| Blank / white detail pane when nothing is selected | Always show an empty state |
| Interactive widget placed over a foldable hinge | Leave hinge bounds empty |
| Mouse / trackpad not in `MaterialScrollBehavior.dragDevices` | Add them in `main.dart` |

---

## What this skill does NOT do

- It does not rewrite a working layout just to use a specific widget class.
- It does not introduce third-party packages when SDK equivalents exist and are
  sufficient.
- It does not apply every item in the checklist to every project — only what the
  code actually needs.
