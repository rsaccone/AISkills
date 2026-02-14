---
description: Production-grade best practices for ActivityKit and Live
  Activities including architecture, token lifecycle, updates,
  concurrency, UI, and testing.
name: live-activities
tags:
- swift
- activitykit
- live-activities
- ios
- widgetkit
---

# Live Activities (ActivityKit) Engineering Standards

Apply these rules whenever generating, reviewing, or refactoring Live
Activities code.

Assume: - iOS 17+ - Swift 5.9+ - Structured concurrency - SwiftUI-based
UI - Production backend push support

------------------------------------------------------------------------

# 1. Architecture

## Separation of Concerns

Live Activity logic must NOT live in SwiftUI Views.

Create a dedicated manager responsible for: - Starting activities -
Observing push-to-start tokens - Observing activity push token updates -
Updating activities - Ending activities - Handling authorization -
Cancelling monitoring tasks

Views: - Render state only - Perform no networking - Start no async
monitoring loops - Contain no business logic

Prefer a protocol-based manager:

protocol LiveActivityManaging { func start(...) func update(...) func
end(...) }

The concrete implementation owns all ActivityKit interaction.

------------------------------------------------------------------------

# 2. Activity Attributes Design

Rules:

-   Keep attributes small and stable.
-   Static properties belong in the outer struct.
-   Dynamic values belong in ContentState.
-   ContentState must conform to Codable and Hashable.
-   Never include secrets or PII.
-   Avoid large payloads.

Example:

struct MyAttributes: ActivityAttributes { struct ContentState: Codable,
Hashable { var progress: Double var status: String }

    var id: UUID

}

------------------------------------------------------------------------

# 3. Authorization

Always check authorization before requesting:

let info = ActivityAuthorizationInfo() guard info.areActivitiesEnabled
else { return }

Never assume Live Activities are available.

------------------------------------------------------------------------

# 4. Starting Activities

Always handle errors:

do { let activity = try Activity`<MyAttributes>`{=html}.request(
attributes: attributes, contentState: state, pushType: .token ) } catch
{ // handle failure gracefully }

Rules:

-   Never force unwrap.
-   Never ignore thrown errors.
-   Do not create duplicate activities unintentionally.
-   Store the returned Activity reference safely.

------------------------------------------------------------------------

# 5. Push Token Lifecycle

If using push updates, you must observe:

for await token in activity.pushTokenUpdates { await
sendTokenToBackend(token) }

For push-to-start:

for await token in
Activity`<MyAttributes>`{=html}.pushToStartTokenUpdates { await
sendTokenToBackend(token) }

Rules:

-   Tokens may rotate.
-   Always forward updated tokens to backend.
-   Do not assume tokens are permanent.
-   Cancel monitoring tasks appropriately.
-   Avoid detached tasks unless absolutely necessary.

------------------------------------------------------------------------

# 6. Updating Live Activities

Use:

await activity.update(using: newState)

Rules:

-   Only update when state changes.
-   Avoid high-frequency updates.
-   Debounce rapid updates.
-   Ensure updates occur from correct actor context.
-   Do not update from SwiftUI views.

Avoid: - Update loops - Excessive network-triggered updates - Heavy
computation before update

------------------------------------------------------------------------

# 7. Ending Activities

End cleanly:

await activity.end(dismissalPolicy: .immediate)

Rules:

-   End activities explicitly when business logic completes.
-   Do not leave orphaned activities.
-   Optionally provide a final state before dismissal.
-   Cancel associated monitoring tasks.

------------------------------------------------------------------------

# 8. Concurrency Rules

-   Prefer structured concurrency.
-   Avoid Task.detached unless required.
-   Cancel long-running tasks on deinit.
-   Avoid shared mutable state without actor isolation.

Manager may be:

-   @MainActor if primarily UI-driven OR
-   an actor if protecting shared mutable state

Never mutate collections of Activity objects across threads unsafely.

------------------------------------------------------------------------

# 9. SwiftUI Rendering Rules

Live Activity SwiftUI views must:

-   Be purely declarative.
-   Render only from context.state.
-   Contain no async work.
-   Perform no networking.
-   Avoid heavy computed properties.
-   Be lightweight and fast.

Dynamic Island layouts must be minimal and clear.

------------------------------------------------------------------------

# 10. Preview & Testing

Provide previews with mock state:

#Preview { MyLiveActivityView(...) }

Testing guidance:

-   Abstract networking behind protocols.
-   Mock push token streams with AsyncStream.
-   Do not require backend to test UI.
-   Simulate activity lifecycle changes.

------------------------------------------------------------------------

# 11. Common Failure Modes

Always check for:

-   Not enabling Live Activities entitlement
-   Ignoring token rotation
-   Not observing pushToStartTokenUpdates
-   Force-unwrapping Activity.request
-   Updating too frequently
-   Detached tasks leaking
-   Starting duplicate activities unintentionally
-   Heavy logic in SwiftUI View
-   Forgetting to end activities
-   Not cancelling monitoring loops

------------------------------------------------------------------------

# 12. Review Output Format

When reviewing Live Activity code:

Provide:

1.  Architecture concerns
2.  Concurrency issues
3.  Token lifecycle mistakes
4.  Update frequency risks
5.  UI violations
6.  Corrected sample code

Keep recommendations concise and production-focused.
