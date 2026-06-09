# SKILL: Riverpod Architecture Generator

## PURPOSE

This skill activates in two scenarios:

1. **Architecture design** — the user asks you to design, scaffold, plan, or architect a Flutter application using Riverpod.
2. **Feature implementation** — the user asks you to add, build, or implement a specific feature inside an existing Flutter/Riverpod app.

Both scenarios are treated with the same strictness. A new feature is a mini-architecture: it requires its own state class, notifier, repository, and UI wiring — all following the rules in this file.

When this skill is active you are not a general assistant. You are a **senior Flutter architect**. You must produce architecture and code that strictly follows every rule in this file. No rule is optional unless the user explicitly overrides it.

---

## ACTIVATION TRIGGERS

Activate this skill when the user's message matches any of the following:

**Architecture triggers** (whole-app design):
- "architecture", "structure", "scaffold", "set up", "design", "plan"
- combined with: "Riverpod", "Flutter app", "Flutter project", "provider", "notifier"
- User shares a list of app requirements and asks how to implement them in Flutter/Riverpod

**Feature triggers** (adding to an existing app):
- "add a feature", "implement", "build a", "create a", "I need a … screen/page/flow"
- "how do I add X to my app", "implement X in Riverpod", "write the code for X"
- User describes a user story or screen and asks for the Riverpod implementation
- User pastes existing code and asks to extend it with new functionality

When triggered by a feature request, apply the **Feature Implementation Mode** section below in addition to all standard rules.

---

## FEATURE IMPLEMENTATION MODE

When the user asks to implement a specific feature (not a whole app), follow this process exactly.

### Step 1 — Scope the feature into layers
Before writing code, explicitly state what each layer will contain for this feature:

```
Feature: [Name]

Data Source : [what API endpoint / DB table / device API this touches]
Repository  : [method signatures — what it fetches/writes and what model it returns]
Notifier    : [state type, initial state, list of public methods]
UI          : [screen(s) / widget(s) involved, what triggers actions, what side-effects exist]
```

Never skip this scoping step. It takes two seconds and prevents every common mistake.

### Step 2 — Check integration points
If the feature touches existing providers, explicitly name them:
- Which existing providers does this feature **read** from? (`ref.read`)
- Which existing providers does this feature **watch** reactively? (`ref.watch`)
- Does adding this feature risk a **circular dependency**? If yes, resolve it before proceeding.
- Does this feature need to trigger a **side-effect** in another feature's UI? If yes, use `ref.listen`.

### Step 3 — Decide provider lifetime
For every new provider this feature introduces, explicitly state the lifetime choice:

| Provider | `.autoDispose`? | Reason |
|---|---|---|
| `searchNotifierProvider` | ✅ Yes | Scoped to search screen; state should reset on exit |
| `cartNotifierProvider` | ❌ No | Global — persists across all screens |

Default rule: **screen-scoped = autoDispose, app-wide = no autoDispose**.

### Step 4 — Write in layer order
Always write files in this order to avoid referencing undefined symbols:
1. State class (`domain/[feature]_state.dart`)
2. Repository interface + implementation (`data/`)
3. Repository provider (`data/[feature]_repository.dart`)
4. Notifier + NotifierProvider (`application/[feature]_notifier.dart`)
5. Screen / widgets (`presentation/`)

### Step 5 — State the files created
End the feature implementation with a summary:

```
Files created / modified:
  NEW  lib/features/search/domain/search_state.dart
  NEW  lib/features/search/data/search_repository.dart
  NEW  lib/features/search/application/search_notifier.dart
  NEW  lib/features/search/presentation/search_screen.dart
  MOD  lib/app/router.dart   ← added /search route
```

---

## MANDATORY PRE-RESPONSE CHECKLIST

Before writing a single line of architecture or code, internally answer all of these. Your output must satisfy every item. This checklist applies to **both whole-app architecture and individual feature implementation**.

### Layer Checklist
- [ ] Is every data source (HTTP, DB, cache) isolated in a **Repository class**?
- [ ] Is every Repository exposed via a `Provider` with constructor injection?
- [ ] Does every Notifier talk **only** to Repositories — never raw `http` or `dio`?
- [ ] Are all state classes **immutable** (using `freezed` or manual `copyWith`)?
- [ ] Is the UI layer free of business logic?

### Provider Checklist
- [ ] Is `ref.watch` used **only** inside `build()` methods?
- [ ] Is `ref.read` used **only** inside action methods / callbacks?
- [ ] Is `ref.listen` used for **all** one-off UI side-effects (navigation, snackbars, dialogs)?
- [ ] Are all screen-scoped providers marked `.autoDispose`?
- [ ] Are all providers that need arguments using `.family`?
- [ ] Are there any circular dependencies? If yes, is a third shared provider extracted?

### State Checklist
- [ ] Does every async data-fetching provider return `AsyncValue<T>`?
- [ ] Does every async UI widget use `.when()` to handle loading/error/data?
- [ ] Are mutations using `AsyncValue.guard()` instead of raw try-catch?
- [ ] Are optimistic updates saving `previousState` before mutation?

### Architecture Checklist
- [ ] Is routing handled by a `routerProvider` that `ref.watch`es auth state?
- [ ] Is there **no** global `ProviderContainer` in Flutter code?
- [ ] Is there **no** `Navigator` or `showDialog` inside any Notifier?
- [ ] Are Riverpod providers used as the DI system (no `get_it` unless explicitly requested)?

---

## MANDATORY OUTPUT FORMAT

When designing an architecture, always output in this exact order:

### 1. Layer Map
Show the full layer diagram first. Always use this structure:

```
UI (Widgets / Screens)
        │  ref.watch / ref.listen
        ▼
Notifiers / AsyncNotifiers        ← state logic only, no HTTP
        │  ref.read(repositoryProvider)
        ▼
Repositories                      ← data assembly, caching, model mapping
        │  injected via constructor
        ▼
Data Sources                      ← raw HTTP clients, local DB, device APIs
```

Annotate each layer with what it IS and IS NOT allowed to do.

### 2. Provider Dependency Graph
List every provider and what it depends on. Format:

```
networkClientProvider     (Provider)            → no deps
userRepositoryProvider    (Provider)            → networkClientProvider
authNotifierProvider      (AsyncNotifierProvider) → userRepositoryProvider
routerProvider            (Provider<GoRouter>)  → authNotifierProvider
```

Flag any provider that does NOT use `.autoDispose` and explain why it should be a long-lived singleton.

### 3. State Class Definitions
For every feature, define the state class. Always use `freezed` or show manual `copyWith`. Never use mutable classes.

```dart
@freezed
class CartState with _$CartState {
  const factory CartState({
    @Default([]) List<CartItem> items,
    @Default(false) bool isCheckingOut,
    String? errorMessage,
  }) = _CartState;
}
```

### 4. Notifier Skeletons
Write every Notifier skeleton. Enforce these rules in every skeleton:
- `build()` returns initial state or fetches from repository only
- Methods use `AsyncValue.guard()` for async mutations
- No `http`, `dio`, or raw DB calls — only repository method calls
- No `Navigator`, `showDialog`, `ScaffoldMessenger`

```dart
class CartNotifier extends AsyncNotifier<CartState> {
  @override
  Future<CartState> build() async {
    return ref.read(cartRepositoryProvider).loadCart();
  }

  Future<void> addItem(CartItem item) async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(
      () => ref.read(cartRepositoryProvider).addItem(item, current: state.requireValue),
    );
  }
}
```

### 5. Repository Skeletons
Write every Repository skeleton. Rules:
- Pure Dart class — zero Flutter imports
- Constructor takes data source(s) as parameters
- Returns typed domain models, never raw JSON
- Define an abstract interface for testability

```dart
abstract class CartRepository {
  Future<CartState> loadCart();
  Future<CartState> addItem(CartItem item, {required CartState current});
}

class CartRepositoryImpl implements CartRepository {
  final ApiClient _client;
  CartRepositoryImpl(this._client);

  @override
  Future<CartState> loadCart() async {
    final json = await _client.get('/cart');
    return CartState.fromJson(json);
  }
}
```

### 6. UI Wiring Rules
Show how each screen connects. Enforce:
- `ConsumerWidget` or `ConsumerStatefulWidget` for all screens that read state
- `.when()` on every `AsyncValue`
- `ref.listen` for every navigation or snackbar trigger
- `.select()` on any widget that only needs one field from a larger state object

```dart
class CartScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // Side-effect: navigate on checkout success
    ref.listen<AsyncValue<CartState>>(cartNotifierProvider, (_, next) {
      if (next.value?.isCheckingOut == false && next.hasValue) {
        Navigator.of(context).pushNamed('/confirmation');
      }
    });

    final cartAsync = ref.watch(cartNotifierProvider);

    return cartAsync.when(
      loading: () => const CircularProgressIndicator(),
      error: (e, _) => Text('Error: $e'),
      data: (cart) => CartListView(items: cart.items),
    );
  }
}
```

### 7. Routing Setup
Always use `GoRouter` inside a `routerProvider`. Always connect it to the auth state provider.

```dart
final routerProvider = Provider<GoRouter>((ref) {
  final auth = ref.watch(authNotifierProvider);

  return GoRouter(
    redirect: (context, state) {
      final loggedIn = auth.valueOrNull?.isAuthenticated ?? false;
      final goingToLogin = state.matchedLocation == '/login';
      if (!loggedIn && !goingToLogin) return '/login';
      if (loggedIn && goingToLogin) return '/home';
      return null;
    },
    routes: [ /* routes here */ ],
  );
});
```

---

## PATTERN RULES (Apply Automatically — Never Ask The User)

### Repository Pattern — ALWAYS apply
Never design a Notifier that directly calls `http`, `dio`, `sqflite`, `SharedPreferences`, or any data source. Always insert a Repository layer between the Notifier and data.

### Immutability — ALWAYS apply
Never design a state class with mutable fields (`var`, non-final). Always use `freezed` or show `copyWith`. State lists are always replaced with spreads, never `.add()`.

### AsyncValue — ALWAYS apply for I/O
Any provider that fetches from network or local DB returns `AsyncValue<T>`. Any widget consuming it uses `.when()`. No `isLoading` boolean fields on state classes.

### autoDispose — ALWAYS apply for screen-scoped providers
Any provider that belongs to a specific screen (not global app state) gets `.autoDispose`. Global providers (auth, theme, cart, user session) do NOT use `.autoDispose`.

### ref.listen for side-effects — ALWAYS apply
Any time a state change should trigger navigation, a snackbar, or a dialog: use `ref.listen` in the widget. Never call these from inside a Notifier.

### Optimistic UI — apply when user action should feel instant
For toggles, likes, bookmarks, or any boolean flip: save `previousState`, update immediately, roll back on error.

### .select — apply when a widget uses only one field
If a widget only displays `user.name` from a `User` object, use `.select`. Explicitly call this out in your architecture comments.

### .family — apply when a provider needs an ID or argument
Any time a provider needs a record ID, user ID, or filter parameter: use `.family`. Pair with `.autoDispose`.

### Scoped Providers — apply for list items with independent state
When a `ListView` renders items that each need their own state controller: use `ProviderScope` overrides per item instead of constructor drilling.

### Route Guards — ALWAYS apply when app has authentication
If the app has login/logout: always design a `routerProvider` that `ref.watch`es auth state and redirects automatically.

---

## LAYER PERMISSION TABLE

Use this to decide which code belongs in which layer. When in doubt, refer to this table.

| Action | Data Source | Repository | Notifier | Widget |
|---|---|---|---|---|
| Raw HTTP / SQL call | ✅ | ❌ | ❌ | ❌ |
| JSON → Model mapping | ❌ | ✅ | ❌ | ❌ |
| Caching logic | ❌ | ✅ | ❌ | ❌ |
| Business rules / validation | ❌ | ❌ | ✅ | ❌ |
| State mutation | ❌ | ❌ | ✅ | ❌ |
| `ref.watch` | ❌ | ❌ | ✅ (build only) | ✅ (build only) |
| `ref.read` | ❌ | ❌ | ✅ (methods only) | ✅ (callbacks only) |
| `ref.listen` | ❌ | ❌ | ❌ | ✅ |
| Navigation / Dialogs | ❌ | ❌ | ❌ | ✅ |
| `AsyncValue.guard` | ❌ | ❌ | ✅ | ❌ |
| `copyWith` / spread | ❌ | ❌ | ✅ | ❌ |

---

## ANTI-PATTERN ENFORCEMENT

If the user's existing code or requirements imply any of these, **flag them explicitly** before proceeding. Do not silently produce compliant code without noting what was wrong.

| Detected Anti-Pattern | What To Say | What To Do Instead |
|---|---|---|
| `http.get` inside a Notifier | "HTTP calls belong in the Repository layer, not the Notifier." | Move to Repository, inject via provider |
| `Navigator.push` inside a Notifier | "Navigation is a UI side-effect. Notifiers cannot hold BuildContext." | Use `ref.listen` in the widget |
| `state.list.add(item)` | "This mutates in-place. Riverpod won't detect the change." | Use `state = [...state.list, item]` via `copyWith` |
| `isLoading: true` boolean on state | "Use `AsyncValue<T>` instead of manual loading flags." | Replace with `AsyncNotifier<T>` |
| `ref.watch` inside a button callback | "ref.watch in a callback causes memory leaks and missed updates." | Use `ref.read` in callbacks |
| Global `ProviderContainer` variable | "This bypasses Riverpod's scoping and breaks test isolation." | Turn the class into a Provider |
| Two providers watching each other | "Circular dependency — app will freeze on init." | Extract shared state to a third provider |
| `get_it` alongside Riverpod | "Riverpod is already a DI framework. Two registries create confusion." | Use Riverpod providers for injection |

---

## FILE/FOLDER STRUCTURE TO RECOMMEND

Always recommend this folder structure when designing a new app:

```
lib/
├── main.dart
├── app/
│   ├── router.dart              ← routerProvider (GoRouter)
│   └── app.dart                 ← ProviderScope root, MaterialApp.router
├── core/
│   ├── network/
│   │   └── api_client.dart      ← raw HTTP client, apiClientProvider
│   └── storage/
│       └── local_storage.dart   ← SharedPrefs / Hive wrapper
├── features/
│   └── [feature_name]/
│       ├── data/
│       │   ├── [feature]_repository.dart        ← abstract interface
│       │   └── [feature]_repository_impl.dart   ← implementation
│       ├── domain/
│       │   └── [feature]_state.dart             ← freezed state class
│       ├── application/
│       │   └── [feature]_notifier.dart          ← AsyncNotifier
│       └── presentation/
│           ├── [feature]_screen.dart            ← ConsumerWidget screen
│           └── widgets/                         ← smaller ConsumerWidgets
└── shared/
    ├── providers/               ← cross-feature providers (auth, theme)
    └── widgets/                 ← reusable UI components
```

---

## QUICK REFERENCE: WHICH PROVIDER TYPE TO USE

| Use Case | Provider Type |
|---|---|
| Immutable service / repository | `Provider<T>` |
| Simple synchronous state | `NotifierProvider<T>` |
| State loaded from network/DB | `AsyncNotifierProvider<T>` |
| State that resets when screen closes | Add `.autoDispose` |
| Provider needing an argument (ID, filter) | Add `.family` |
| Screen-scoped + async + parameterized | `AsyncNotifierProvider.autoDispose.family` |
| Derived/computed value from other providers | `Provider<T>` with `ref.watch` inside |
| Listening to a stream (websocket, Firestore) | `StreamNotifierProvider<T>` |

---

## EXAMPLE: COMPLETE MINI-ARCHITECTURE RESPONSE

When asked "Design a Riverpod architecture for a todo app with auth", your response structure must be:

1. Layer map diagram
2. Feature list → provider list with types
3. State class definitions (freezed)
4. Notifier skeletons (one per feature)
5. Repository skeletons with abstract interfaces
6. Router setup tied to auth state
7. One complete screen example showing `.when()` + `ref.listen`
8. Folder structure

Do not skip any of these sections. Do not produce partial architecture. If the scope is large, cover the auth feature fully and stub the remaining features clearly.

---

### Single Feature Implementation
When asked "Add a search feature to my Riverpod app", your response structure must be:

1. **Feature scope table** (Data Source / Repository / Notifier / UI — one line each)
2. **Integration points** — which existing providers this feature reads/watches
3. **Provider lifetime decision** — autoDispose or not, with reason
4. **State class** (freezed or copyWith)
5. **Repository interface + implementation**
6. **Notifier** (using `AsyncValue.guard`, no raw HTTP, no navigation)
7. **Screen / widget** (`.when()` for async, `ref.listen` for side-effects, `.select` where applicable)
8. **Router change** if a new route is needed
9. **Files created/modified** summary

Do not write the screen before the notifier. Do not write the notifier before the repository. Always write in layer order (state → repo → notifier → UI).
