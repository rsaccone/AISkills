---
name: swiftui-best-practices
description: Apply SwiftUI best practices: predictable state, small views, testable logic, and correct navigation patterns.
tags: [swiftui, mvvm, architecture, ui, refactor]
---

# SwiftUI Best Practices

Follow these rules when authoring SwiftUI. Prefer modern patterns unless constrained by OS targets.

## State & data flow
- Keep SwiftUI views **stateless where possible**.
- Put business logic in a ViewModel or model layer; views bind to **simple state**.
- Prefer one clear source of truth; avoid duplicating state (e.g., `@State` mirroring model fields) unless it’s an intentional edit buffer.
- Avoid passing bindings deep through many layers; consider a small view model, environment objects, or feature state.

## View composition
- If a `body` grows large, extract subviews:
  - Use a **private nested View struct** for substantial UI chunks.
  - Use a **computed var** only for small, local snippets (roughly < 15 lines).
- Keep modifiers close to the view they affect; avoid huge modifier chains by extracting view builders.

## Identity & lists
- Use stable identity in lists: `ForEach(items, id: \.id)` or `Identifiable`.
- Avoid using indices as IDs unless the list is truly static.
- Don’t generate new UUIDs in `body` for identity.

## Navigation
- Prefer `NavigationStack` (and `NavigationSplitView` on iPad/macOS) over legacy APIs.
- Avoid pushing navigation state into many views; keep navigation decisions close to the feature boundary.
- Prefer typed navigation (routes as enums / Hashable) rather than stringly-typed paths.

## Performance pitfalls
- Keep `body` pure; do not start async work directly in `body`.
- Use `.task(id:)` for async work that depends on changing inputs.
- Avoid heavy work in computed properties that run during rendering.
- Use `@State` for local UI state; avoid `@ObservedObject` unless required by legacy patterns.

## Animations
- Prefer explicit animations near the state change (`withAnimation { ... }`) rather than global `.animation(...)` modifiers.
- Use `.transaction` sparingly, with intent.

## Accessibility
- Provide meaningful labels for icons and custom controls.
- Ensure tappable areas are sufficient (contentShape / padding where needed).
- Don’t rely on color alone to convey meaning.

## Output format (when generating SwiftUI code)
- Provide complete, compiling SwiftUI code.
- If introducing a ViewModel, keep it minimal and test-friendly.
- Add brief comments only where behavior is non-obvious.
