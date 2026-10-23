# Synchronous Transform Overlay

[🏠 Home](../../../../../../../__docs__/README.md)

The synchronous transform overlay (`SynchronousTransformOverlay`) keeps the
layout queries of the renderer (`measure`, `measureInWindow`,
`getBoundingClientRect`, `findNodeAtPoint`,
`UIManager::getRelativeLayoutMetrics`) in sync with views whose `transform` was
changed _synchronously on the UI thread_, i.e. without a ShadowTree commit.

It is enabled by the `enableSynchronousTransformOverlay` feature flag.

## 🚀 Usage

Nothing changes for application code: the layout APIs (`measure()`,
`getBoundingClientRect()`, ...) simply start returning the position of the
mounted view when the view was moved by a native-driven animation. The most
visible effect is that `Pressable` (and the `Touchable*` components), which
validate an ongoing press against a rectangle obtained from `measure()`, keep
firing `onPress` for views that are being animated with `Animated`
(`useNativeDriver: true`), sticky headers, or third-party animation libraries
that use the same mechanism.

Animation systems that write props synchronously through the platform entry
points (`FabricUIManager.synchronouslyUpdateViewOnUIThread` on Android,
`RCTSurfacePresenter`'s `schedulerDidSynchronouslyUpdateViewOnUIThread:props:`
on iOS, `UIManager::synchronouslyUpdateViewOnUIThread` /
`synchronouslyUpdateAnimatedProps` in C++) do not need to do anything either:
the overlay is fed from those entry points.

Systems that bypass them (for example by writing to the mounting manager
directly) should call `UIManager::recordSynchronousTransform(tag, props)` with
the props they applied, and `UIManager::removeSynchronousTransform(tag)` when
they stop managing a view so that the committed props become authoritative
again.

## 📐 Design

### The problem

Fabric assumes that the committed ShadowTree describes what is on the screen.
Animation systems (`Animated` with the native driver, the shared Animation
Backend, Reanimated's synchronous props path, ...) break that assumption on
purpose for non-layout props: committing a new ShadowTree revision on every
animation frame means cloning nodes, running layout and diffing, which is too
expensive on low-end devices. Instead they apply those props straight to the
mounted views (`synchronouslyUpdateViewOnUIThread`) and only commit the final
values when the animation settles (or when a React commit happens, through a
commit hook).

Most non-layout props are harmless in that respect, but `transform` changes
where a view is, and the geometry of the ShadowTree is what
`LayoutableShadowNode::computeRelativeLayoutMetrics` reads
(`ConcreteViewShadowNode::getTransform` resolves the committed `transform`
prop). Native hit-testing, on the other hand, uses the real view hierarchy and
sees the live transform. Anything that mixes the two is inconsistent during the
animation: `Pressability` measures the responder region with `measure()` (stale
ShadowTree position) and compares it with the touch coordinates reported by the
platform (real position), so a press with a slight finger movement leaves the
"stale" rectangle and `onPress` is dropped.

### The overlay

`UIManager` owns a `SynchronousTransformOverlay`: a thread-safe map from view
tag to the last `transform` / `transformOrigin` that was written synchronously.
The values are parsed with the same `fromRawValue` conversions used for
ShadowNode props, so an entry is exactly what a commit of those props would have
produced.

The overlay implements `LayoutableShadowNode::TransformOverrideProvider`, a
small interface defined next to `LayoutInspectingPolicy`. When a provider is
present in the policy, `computeRelativeLayoutMetrics` and `findNodeAtPoint` ask
it for the transform of every node they visit and fall back to `getTransform()`
when the provider has no entry (`resolveTransform`). Entries are resolved
against the node's committed layout metrics the same way
`ConcreteViewShadowNode::getTransform` does (percentages, `transformOrigin`),
with the committed props filling in whichever half of the (`transform`,
`transformOrigin`) pair was not written synchronously, which is also what the
mounted view shows.

`UIManager::getTransformOverrideProvider()` returns the overlay when the feature
flag is enabled and the overlay is not empty, and `nullptr` otherwise, so the
hot path of the layout queries only pays for the lookups while something is
actually animating. `UIManager::getRelativeLayoutMetrics` and
`UIManager::findNodeAtPoint` inject it automatically; `NativeDOM` and
`UIManagerBinding` pass it to the `dom::` helpers.

### Recording

Every synchronous write funnels through one platform-specific entry point, which
records the transform before applying it to the view:

- Android: `FabricUIManager.synchronouslyUpdateViewOnUIThread` (legacy Animated
  and the single-view path of C++ Animated) copies the `transform` /
  `transformOrigin` entries into a fresh `NativeMap` and calls
  `FabricUIManagerBinding.recordSynchronousTransform`; the batched
  `AnimatedProps` path of the Animation Backend is recorded in
  `FabricUIManagerBinding::schedulerShouldSynchronouslyUpdateAnimatedProps`.
- iOS: `RCTSurfacePresenter`'s
  `schedulerDidSynchronouslyUpdateViewOnUIThread:props:`, which is reached by
  `RCTPropsAnimatedNode`, the C++ scheduler delegate and third-party libraries
  alike.
- `ReactCxxPlatform`: `SchedulerDelegateImpl`.

### Lifetime of an entry

An entry mirrors the state of the mounted view: once a transform has been
written synchronously, the platforms keep showing the latest synchronous value
even across React commits (`tagToSynchronousMountProps` on Android,
`propKeysManagedByAnimated` on iOS). The overlay therefore keeps the entry
until:

- the system that wrote it stops managing the view and calls
  `UIManager::removeSynchronousTransform` (the Animation Backend does so when it
  clears its `AnimatedPropsRegistry` after the `AnimationEndSync` commit that
  carries the final values into the ShadowTree), or
- the view is deleted. Deletions are not observed directly; instead, once the
  overlay grows past a threshold, `UIManager::shadowTreeDidCommit` walks the
  current revisions of all surfaces and drops the entries whose tags no longer
  exist (`prune`). The threshold doubles after each prune so that a steady set
  of animated views does not cause a tree walk on every commit.
  `UIManager::stopSurface` prunes unconditionally.

### What is not covered

- Layout props always go through a commit, so they never need the overlay.
- Props other than `transform` do not affect geometry and are ignored.
- The overlay describes the mounted views; if a platform masks a React commit
  with a stale synchronous value (see "Lifetime" above), the overlay
  intentionally agrees with the screen rather than with the ShadowTree.

## 🔗 Relationship with other systems

### Part of

- UIManager (`react/renderer/uimanager`).

### Used by this

- `LayoutableShadowNode` (`react/renderer/core`): the
  `TransformOverrideProvider` interface and `LayoutInspectingPolicy`.
- `BaseViewProps::resolveTransform` and the `conversions.h` parsers
  (`react/renderer/components/view`) to resolve entries like committed props.
- The feature flags system (`enableSynchronousTransformOverlay`).

### Uses this

- The DOM layout APIs (`react/renderer/dom`, `NativeDOM`, `UIManagerBinding`):
  `measure`, `measureInWindow`, `getBoundingClientRect`, `findNodeAtPoint`.
- `UIManager::getRelativeLayoutMetrics`, and through it `PointerEventsProcessor`
  (pointer capture retargeting) and libraries such as Reanimated's `measure()`.
- The platform mounting layers and the
  [Animation Backend](../../animationbackend/__docs__/AnimationBackend.md),
  which feed and clear it.
