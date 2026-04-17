# React Native — Language-Specific Review Hints

Auto-loaded when the diff contains `.tsx` or `.ts` files.

These are patterns that have been flagged on real PRs in this codebase. Inject these into all zombie agents when dispatched.

---

## Component lifecycle and hooks

**`useEffect` cleanup:**
- Effects that set up subscriptions, listeners, or timers must return a cleanup function. No cleanup = memory leak on unmount.

**`useCallback` / `useMemo` dependencies:**
- Stale closure bugs: a callback that captures a value from scope but doesn't list it in deps will use the stale value.
- Over-memoization is also a smell — flagging `useMemo` on a primitive or a simple object literal is noise.

**`useRef` for values that shouldn't trigger re-renders:**
- Mutable values that track state across renders without causing re-renders (e.g., reconnect timers, abort controllers) should be `useRef`, not `useState`.

---

## Navigation

**`useNavigation` typing:**
- Should be typed with the screen's specific param list, not the generic `NavigationProp`.
- Nav params accessed with non-null assertion (`route.params.foo!`) is acceptable team convention.

**Screen registration:**
- New screens added to a navigator must be registered in the navigator's param list type.

---

## Network and async state

**`isOffline` derivation:**
- `isOffline` should be derived from `!isInternetReachable && isConnected !== null`. A plain `!isConnected` can return false positives on Android.

**`ScreenQueryResult` vs `QueryResult`:**
- Use `ScreenQueryResult` for screen-level data queries. `QueryResult` for component-level.

---

## Styling

**`StyleSheet.create` on new components:**
- New components should use NativeWind `className` + `cn()`. `StyleSheet.create` on new components is a convention deviation.

**Inline styles on new components:**
- Inline style objects recreate on every render. Flag them unless they depend on dynamic values that can't be expressed as className.

---

## Expo / OTA

**`RUNTIME_VERSION` in `app.config.ts`:**
- Bumping `RUNTIME_VERSION` blocks OTA updates for existing users until the next store release. This should be intentional and noted in the PR description.
- If the PR introduces feature-flagged behavior AND bumps `RUNTIME_VERSION`, it should probably be two PRs.
