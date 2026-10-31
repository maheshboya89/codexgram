# Codexgram v1 Implementation Plan

**Status:** Product and technical plan; implementation has not started.

## 1. Product goal and release scope

Build a clean, minimal, mobile-first social app for iOS and Android. The first release is an invite-only beta in India, in English, for people aged 13 and over. The product is a general-purpose Instagram-style app; v1 has no payments, ads, or broad analytics.

### Included in v1

- Google sign-in on iOS and Android; Apple sign-in on iOS. Google and Apple are the only sign-in methods.
- One-time invite codes for beta access.
- Profiles with a unique username, display name, bio, avatar, optional website, and public/private setting. New profiles default to public.
- Image or short-video posts, follows and private follow requests, likes, comments, direct messages, activity notifications, blocking, reporting, and self-service account deletion.
- A four-tab app shell: **Home**, **Messages**, **Explore**, **Profile**.
- **Custom navigation:** a custom-designed bottom tab bar, implemented with Expo Router. Do not use the platform-native tab bar.
- **Light theme only.** No dark theme in v1. Use semantic color tokens, but leave exact palette and visual styling open until the user supplies a design reference.

### Explicitly out of scope

- Web app, group chats, carousels, video longer than 60 seconds, email/password or phone sign-in, subscriptions, ads, and offline write queues.
- A design reference or final color palette; these will be supplied later.

## 2. Current project and technical constraints

- Existing project: Expo SDK `58.0.22`, React Native `0.88.1`, Expo Router `58.0.29`, TypeScript, npm lockfile, and route files under `src/app/`.
- Keep Expo Router file-based navigation. Screens and layouts belong in `src/app/`; reusable UI, hooks, and Convex-independent utilities belong outside the route directory.
- Preserve Continuous Native Generation. Do not hand-create or edit `ios/` or `android/`; configure native behavior through `app.json` and config plugins.
- Use NativeWind as requested. Before adding it, verify a **stable** release on this exact Expo SDK. NativeWind v5 is currently a pre-release and should not be used for this production-bound beta. If stable NativeWind compatibility with SDK 58 cannot be verified, stop and report the compatibility issue rather than silently changing the styling stack. [NativeWind v5 status and target](https://www.nativewind.dev/v5/getting-started/installation)
- Use `npx expo install` for Expo and native dependencies. The project uses npm; there is no `bun.lock`.
- Native Google/Apple sign-in and push notifications require development builds. Plan on EAS development and production builds with iOS and Android credentials; Expo Go is not the validation target. [Clerk Expo auth](https://clerk.com/docs/expo/getting-started/quickstart), [Expo push setup](https://docs.expo.dev/push-notifications/push-notifications-setup/)

### Compatibility gate before feature work

1. Confirm the current stable `@clerk/expo` peer range and native Google/Apple flows support Expo SDK 58. Current Clerk references found during planning explicitly describe compatibility through earlier SDKs, so treat SDK 58 support as unverified until checked against the package and current official docs. [Clerk versioning](https://clerk.com/docs/guides/development/upgrading/versioning)
2. Confirm stable NativeWind and its required Metro/Babel/Tailwind setup against Expo SDK 58.
3. Run a minimal iOS and Android development build with Clerk providers and the custom tab shell before implementing the social features.
4. If a required library does not support SDK 58, keep Clerk and NativeWind as the requirements and return with a concrete compatibility choice; do not downgrade Expo or replace a requested provider without approval.

## 3. User journeys and behavior

### First launch and authentication

1. Show a light-mode welcome/sign-in screen with Google and Apple options (Apple only on iOS).
2. Require a valid, unused invite code before account access. Validate the code first, but consume it atomically only after successful authentication and profile creation so an abandoned OAuth flow does not burn a code.
3. After Clerk authentication, require a unique username. Allow display name, bio, avatar, website, and public/private choice to be completed in profile setup; default privacy is public.
4. Require an age gate for 13+. Do not permit under-13 accounts.
5. Returning users restore their Clerk session and enter the app. Route guards must wait for auth state before redirecting to sign-in or app routes.

### Home and profiles

- Home shows followed accounts' visible posts newest first. Suggested people appear in a separate section and never introduce posts the user cannot access.
- Public account follows are accepted immediately. Following a private account creates a pending request; the recipient can accept or decline. Users can cancel pending requests and unfollow accepted follows.
- A public-to-private change immediately restricts all of that user's existing posts to approved followers. A private profile may appear in people search, but its posts and interactions must never be returned to an unapproved viewer.
- Any signed-in user who can view a post may like or comment on it. One like per user per post; tapping like again removes it. Users may delete their own posts and comments.
- The Profile tab shows the current user's profile and edit/settings routes. Other users' profiles open from feeds, search, comments, and message threads.

### Create and view posts

- Select one existing photo or video from the device library; no in-app camera capture in v1.
- Allow an optional caption. One media item per post. Enforce a 15 MB photo cap, 50 MB video cap, and 60-second video duration cap before upload and again in backend validation where possible.
- Store media in Convex Storage and post metadata in Convex. Upload through generated upload URLs; persist the storage ID only after successful upload. Clean up uploaded media if post creation fails or the post is later deleted.
- Public posts can appear in Explore. Private posts are only returned to approved followers. Explore never exposes private post media, counts, comments, or likes to unapproved viewers.

### Explore and search

- Search people and public posts. People search may return public and private profiles, but the profile privacy state must be respected when opening them.
- Public posts are browseable in Explore; posts from private accounts are excluded from Explore results.
- Debounce search input, show loading/empty/error states, and paginate results. Search indexes and queries must filter visibility on the backend, not only in the client.

### Messages

- One-to-one conversations support text and image messages. Each image follows the 15 MB photo cap; no video attachments.
- A user may start a conversation when either user has an accepted follow relationship with the other. Once created, the conversation remains active after an unfollow.
- Show unread counts and read status. Mark messages read when their conversation is opened. Use Convex reactive subscriptions for conversation and message updates.
- A block prevents new messages and hides the conversation and its messages from both participants while the block is active.

### Activity, push, and network failures

- Store an in-app activity inbox for new messages, follow requests, likes, and comments. Support read/unread state and per-category push preferences.
- Register and remove device push tokens. Request OS permission in context; if permission is denied or push delivery fails, the in-app inbox remains available.
- Require network for writes. Show retry actions for failed posts, comments, follows, and messages; do not queue social writes offline. Render cached data when available.

### Blocking, reports, and account deletion

- Blocking removes accepted follows and pending requests in both directions, prevents new follows/messages, and hides profiles, posts, and conversations from each other. Keep underlying conversation records for account-deletion retention, but deny access while blocked.
- Users can report profiles, posts, comments, or messages. A protected internal admin queue lets admins review, dismiss, hide content, or suspend accounts. Admin access is enforced in Convex functions, not just route visibility.
- Self-service account deletion removes the profile, posts, comments, likes, follow relationships, push tokens, and media. Preserve existing chat history for other participants and replace the deleted sender's identity with a deleted-user label. Run large cleanup work asynchronously and show deletion progress/confirmation.

## 4. Data model and authorization boundaries

Clerk is the source of truth for authentication identity. Convex owns app profiles, social graph, content, messages, notification state, and moderation. Store the Clerk subject ID on the Convex user profile; do not trust client-supplied user IDs for ownership.

| Entity | Key fields and constraints |
| --- | --- |
| `users` | `clerkId`, normalized unique `username`, display name, bio, avatar storage ID, website, `privacy` (`public`/`private`), role (`user`/`admin`), age-gate result, notification preferences, created/updated timestamps. |
| `inviteCodes` | Hashed code, creator, redeemed-by user, created/redeemed timestamps, one-time redemption. |
| `posts` | Author, storage ID, media type, optional caption, video duration when applicable, created timestamp, deleted state. Post visibility is derived from the author's current privacy and accepted follower relationships. |
| `follows` | Follower, followed user, status (`pending`/`accepted`), timestamps; unique directed pair. Public follows become accepted immediately. |
| `likes` | User, post, timestamp; unique user/post pair. |
| `comments` | Author, post, text, timestamp, deletion state. Only the author may delete their comment. |
| `conversations` | Two participant IDs, created/last-message timestamps, per-participant read state. No group conversations. |
| `messages` | Conversation, sender, optional text, optional image storage ID, created timestamp, read timestamp/state. Preserve history when an account is deleted, with sender identity removed. |
| `notifications` | Recipient, actor, type, target entity, created/read timestamps. |
| `pushTokens` | User, device token, platform, last-seen timestamp; remove invalid tokens. |
| `blocks` | Blocker, blocked user, timestamp; unique directed pair. |
| `reports` | Reporter, reported entity, reason, status, review metadata, timestamps. |
| `appSettings` | Admin-controlled `uploadsPaused` flag and any operational limits needed to protect the monthly budget. |

### Server-side authorization invariants

- Every Convex query and mutation derives the current user from the authenticated Clerk token.
- Private visibility is checked in Convex for posts, likes, comments, profile details, search results, and media URL access. Hiding a component in React Native is not authorization.
- Mutations validate post/comment/message ownership, follow eligibility, block status, post visibility, media type/size/duration, and uniqueness constraints.
- On privacy changes, visibility takes effect immediately for reads; no copied public visibility flag may bypass the owner's current privacy setting.
- On block, reject new social actions in both directions and filter existing data from both users' queries.

## 5. Architecture and UI structure

### App routing and UI

- Keep route screens in `src/app/`. Add auth, onboarding, a protected app group, nested stacks for tab flows, create-post, conversation detail, profile detail/edit, notification inbox, and a protected admin route.
- Implement four Expo Router tabs with a custom tab bar. Keep navigation state and deep links in Expo Router; make the visual tab bar custom and accessible (labels, selected state, safe-area handling, and large touch targets).
- Use NativeWind for styling once SDK 58 compatibility is verified. Establish semantic light-theme tokens for background, surface, text, muted text, border, and accent. Do not finalize token values until the design reference arrives. Force light appearance at app configuration and remove automatic dark-mode branching.
- Use existing `expo-image` for images. Add SDK-compatible Expo media selection, video playback, and notifications through `npx expo install`; follow SDK 58 docs. [SDK 58 reference](https://docs.expo.dev/versions/v58.0.0/), [ImagePicker](https://docs.expo.dev/versions/v58.0.0/sdk/imagepicker/), [Notifications](https://docs.expo.dev/versions/v58.0.0/sdk/notifications/)
- Use virtualized lists and pagination for feeds, search, activity, and messages. Pause video playback for off-screen posts and avoid loading full-resolution media into list cells.

### Backend and integrations

- Clerk handles Google/Apple authentication and session lifecycle. Convex validates Clerk JWTs and provides authenticated queries/mutations through its Clerk provider. [Convex + Clerk](https://docs.convex.dev/auth/clerk)
- Convex handles database, indexes/search, reactive updates, file storage, server-side access checks, scheduled cleanup, and notification event creation. [Convex realtime](https://docs.convex.dev/realtime), [Convex file uploads](https://docs.convex.dev/file-storage/upload-files)
- Expo Notifications handles device token registration and push delivery. Push is best-effort; Convex inbox state is durable and authoritative.
- Store only public client configuration in `EXPO_PUBLIC_*` variables. Keep Clerk secrets and service credentials in Clerk, Convex, or EAS secret storage; never commit them. Keep separate dev and production Clerk/Convex deployments.

### Cost guard

- Monthly service budget: under $100. Beta target: more than 1,000 invited users. Current Convex Free/Starter published allowances include 1 GB storage and 1 GB monthly egress; overages can accrue, so re-check current pricing before launch. [Convex limits](https://docs.convex.dev/production/state/limits)
- Add an admin-controlled `uploadsPaused` switch enforced by every media-upload mutation. Monitor Convex storage/egress usage and pause uploads before the cap is reached. Keep text chat, browsing, and other non-upload actions available when paused.
- Treat the upload pause as a conservative operational control, not an automatic billing guarantee. Configure provider billing alerts and review usage during the beta.

## 6. Step-by-step implementation sequence

### Phase 0 — Compatibility and service setup

1. Verify stable NativeWind and Clerk Expo compatibility with Expo SDK 58. Record exact package versions from official docs/package metadata.
2. Create Clerk dev/prod apps, enable Google and Apple, configure OAuth client IDs, bundle/package identifiers, URL schemes, and required native plugins.
3. Create Convex dev/prod deployments and configure Clerk issuer/JWT integration.
4. Configure EAS development builds and push credentials for iOS and Android. Keep native projects generated through Expo CNG.
5. Build a thin auth-and-navigation proof of concept on both platforms. Do not move to feature work until Google, Apple, Convex auth, NativeWind styling, and the custom tab shell work in development builds.

### Phase 1 — App foundation

1. Add Clerk and Convex providers to the root layout with secure session token storage and clear loading/error states.
2. Add route groups and guards for signed-out, invite/onboarding, authenticated, and admin flows.
3. Add the light-only theme tokens, reusable layout primitives, and custom four-tab bottom bar.
4. Add development/production environment examples and document local startup, EAS development builds, and Convex deployment commands.

### Phase 2 — Invite, onboarding, and profiles

1. Implement one-time code validation/redemption and the 13+ age gate.
2. Create/complete the Convex user profile after Clerk sign-in; enforce normalized unique usernames.
3. Build profile display/edit, avatar selection/upload, public/private toggle, settings, and sign-out.
4. Implement self-service deletion entry point and asynchronous deletion workflow before adding dependent features.

### Phase 3 — Social graph and privacy foundation

1. Add Convex schema/indexes and functions for follow, unfollow, pending requests, accept/decline/cancel.
2. Add block/unblock and shared server-side visibility helpers; cover query and mutation authorization.
3. Add backend tests for privacy, relationship, ownership, and block invariants before building feed UIs.

### Phase 4 — Posts and Home

1. Add media picker, client validation, upload progress/cancel/retry, Convex Storage upload, and orphan cleanup.
2. Add post creation, caption, feed queries, post detail, like toggle, comments, and owner delete flows.
3. Implement chronological Home feed, separate suggested-people cards, paginated lists, empty/error states, and visible-item video playback.
4. Ensure public/private switching immediately changes access to all existing posts.

### Phase 5 — Explore and search

1. Add debounced people search and public-post search indexes.
2. Enforce private-post exclusion on server search queries.
3. Build results, profile previews, public post grid/detail, no-results, loading, and retry states.

### Phase 6 — Messages and notifications

1. Add one-to-one conversation creation and messaging eligibility based on accepted follow direction.
2. Implement text/image messages, read state, unread counts, and reactive conversation updates.
3. Implement durable in-app notifications for follow requests, likes, comments, and new messages.
4. Add device token registration, per-category preferences, contextual OS permission prompt, and push actions/deep links.
5. Test blocked-user behavior, unfollow-with-existing-chat behavior, offline errors, token invalidation, and duplicate event prevention.

### Phase 7 — Safety, moderation, and operations

1. Add report flows for profiles/posts/comments/messages with server-side duplicate/rate protections.
2. Add an admin-only review queue and actions to dismiss, hide content, and suspend users.
3. Add admin invite code creation and upload-pause controls.
4. Add crash/backend error monitoring, privacy-conscious logs, usage alerts, and support/runbook notes.

### Phase 8 — Beta hardening and release

1. Run lint and TypeScript checks, then the automated test suite.
2. Verify all primary journeys and error states on iOS and Android development builds.
3. Verify Clerk/Convex production environment separation, EAS credentials, notification delivery, store configuration, and deletion cleanup.
4. Start invite rollout in India, monitor crashes/errors/reports and Convex storage/egress, and pause uploads if the budget guard is reached.

## 7. Verification and acceptance scenarios

- New user redeems a valid invite, signs in with Google/Apple as supported, passes age gate, creates a unique username, and lands on Home. Invalid/redeemed code and cancelled auth do not create a usable profile or consume the code incorrectly.
- Returning user restores session; sign-out returns to auth. Protected app and admin routes cannot be opened by signed-out or non-admin users.
- Public account follows are immediate; private account follows remain pending until accepted. Decline/cancel/unfollow update counts and access correctly.
- A private post is never returned to an unapproved viewer through feeds, search, direct document/media access, or related like/comment queries. Switching public to private revokes access immediately.
- A user can create, like/unlike, comment on, and delete a post within configured media limits. They cannot delete another user's content or interact with a post they cannot view.
- Explore returns public posts and discoverable profiles only; private posts never leak through search or profile routes.
- A conversation can start only after an accepted follow in either direction. Read state updates; conversation remains active after unfollow; a block prevents access and new messages both ways.
- Push permission denial does not remove inbox notifications. Failed push delivery leaves durable in-app activity. Category preferences are honored.
- Reports reach the admin queue; only admins can review or apply moderation actions. Blocking hides the relationship and communication surfaces both ways.
- Account deletion removes user-owned profiles/posts/comments/media and social links, retains recipient-visible chat history with deleted identity, and completes safely when cleanup spans many records.
- When uploads are paused, post and DM image uploads are rejected server-side with a clear message; browsing, text messages, and other non-upload functionality continue.
- Slow network, lost connection, upload failure, empty feeds, search with no results, and media permission denial have useful recovery or retry states.

## 8. Assumptions and unresolved risks

### Assumptions to carry into implementation

- The 13+ age gate collects date of birth for validation but stores only the pass result and timestamp, not the full date of birth.
- Usernames are case-insensitive unique handles. Captions/comments and message text have reasonable server-enforced length limits to be set with the UI implementation.
- DMs are one-to-one and support text or one image attachment per message; message deletion/unsend is not part of v1.
- Admin invite codes are created in a protected admin route. Admin assignment is provisioned from a server-side allowlist/configuration, never by a client-editable profile field.
- Monitoring vendor is not selected; choose one with a free/low-cost tier compatible with Expo SDK 58 before implementation.
- Exact visual colors and component styling remain placeholders until the design reference arrives.

### Risks requiring attention

- **Clerk/Expo SDK 58 support is not yet verified.** Current compatibility material located during planning describes earlier SDK ranges. The compatibility spike is a hard gate; no silent Expo downgrade or auth-provider replacement.
- **NativeWind SDK 58 support needs confirmation.** Avoid NativeWind v5 prerelease for this beta unless the user explicitly accepts prerelease risk.
- **Budget may be tight.** More than 1,000 users, video egress, and Convex's included storage/egress allowances can push costs above $100/month. An operational pause flag and provider alerts reduce, but cannot eliminate, this risk.
- **Media processing is undecided.** The v1 limits are specified, but no transcoding/thumbnail pipeline is selected. Confirm accepted codecs and the SDK 58 thumbnail approach during the media spike; do not add an external media vendor without approval.
- **India and age policy.** The plan implements a 13+ gate and India-first beta, but the product owner must provide the required user terms, privacy notice, and moderation/support process before inviting users.
- **Push delivery is not guaranteed.** Maintain the in-app inbox as the source of truth and monitor provider credentials and invalid tokens.

## 9. Documentation references checked

- [Expo SDK 58 reference](https://docs.expo.dev/versions/v58.0.0/)
- [Expo Router navigation](https://docs.expo.dev/router/introduction/)
- [Clerk Expo quickstart](https://clerk.com/docs/expo/getting-started/quickstart)
- [Convex + Clerk](https://docs.convex.dev/auth/clerk)
- [Convex file upload and storage](https://docs.convex.dev/file-storage/upload-files)
- [Convex current limits](https://docs.convex.dev/production/state/limits)
- [NativeWind v5 release status](https://www.nativewind.dev/v5/getting-started/installation)
