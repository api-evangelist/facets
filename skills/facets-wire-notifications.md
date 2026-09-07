---
name: Wire Facets notifications to a channel
description: Create a notification channel (Slack, PagerDuty, Zenduty, Teams, email or a generic webhook), subscribe a project to the event types you care about, and test it before relying on it.
api: openapi/facets-control-plane-openapi.yml
operations:
  - getAllChannelTypes
  - getAllChannels
  - createNotificationChannel
  - testNotificationChannel
  - getAllNotificationTypes
  - getSubscriptionAttributes
  - getNotificationTagsForNotificationType
  - createSubscription
  - getAllSubscriptions
  - editSubscription
generated: '2026-09-07'
method: generated
source: Grounded in operationIds verified verbatim in openapi/_original/facets-control-plane-openapi.json, plus https://www.facets.cloud/docs/features-and-guides/monitoring-and-observability/notifications.
---

# Wire Facets notifications to a channel

Facets' event surface is subscription-based. A subscription is a unique grouping of
**blueprint (project) + channel + notification type**, optionally narrowed by advanced
filters.

## Know the limits of what is published

Facets ships **no AsyncAPI document and no payload schema** for any notification type. You
can wire delivery reliably; you cannot promise a consumer what the body will look like
without observing one. Say that, rather than describing a payload you have not seen.

The docs also warn that their own list of notification types is not authoritative — the
authoritative list is whatever `getAllNotificationTypes` returns from *this* control plane
(the CLI equivalent is `raptor get notification-types`). Always read it live.

## Steps

1. **List what this control plane supports.** `getAllChannelTypes` —
   `GET /cc-ui/v1/notification/channelTypes`, and `getAllNotificationTypes` —
   `GET /cc-ui/v1/notification/notificationTypes`. Do not work from the docs' list; work from
   these.

2. **Check for an existing channel first.** `getAllChannels` —
   `GET /cc-ui/v1/notification/channels`. Reusing a channel is almost always right; creating
   a duplicate Slack channel entry is a common mess.

3. **Create the channel if needed.** `createNotificationChannel` —
   `POST /cc-ui/v1/notification/channels`. The `NotificationChannel` body carries
   `channelType`, `channelAddress` (the webhook URL for generic webhooks),
   `authorizationHeader`, `integrationKey` (PagerDuty/Zenduty) and `emailAddresses`.

   **`authorizationHeader` and `integrationKey` are credentials.** Never log them, never echo
   them back, never put them in a commit.

4. **Test before subscribing.** `testNotificationChannel` —
   `POST /cc-ui/v1/notification/channels/test`. This sends a mock notification so you can
   confirm delivery without waiting for a real event. Do this every time — it is cheap and it
   is the only way to catch a wrong URL before an incident does.

5. **Discover what you can filter on.** `getSubscriptionAttributes` —
   `GET /cc-ui/v1/notification/notificationType/{notificationType}/attributes` and
   `getNotificationTagsForNotificationType` —
   `GET /cc-ui/v1/notification/{notificationType}/tags`. This is how you build the "only
   Critical severity alerts" style of filter the docs describe.

6. **Create the subscription.** `createSubscription` —
   `POST /cc-ui/v1/stacks/{stackName}/notification/subscriptions` for a project-scoped one,
   or `createSubscription_1` — `POST /cc-ui/v1/notification/subscriptions` for the global
   form. The `Subscription` body carries `channelId`, `notificationType`,
   `notificationSubject`, `filters` and `payloadJson` (the custom payload template for
   generic webhooks).

   Audit Log events can be subscribed **globally**, without a project — that is the one type
   the docs call out as project-independent.

7. **Confirm.** `getAllSubscriptions` —
   `GET /cc-ui/v1/stacks/{stackName}/notification/subscriptions`.

## Volume discipline

Subscribing a channel to every notification type on every project is how a team learns to
ignore the channel. Start from the question the human actually asked — usually deployment
status and alerts for one project — and add types only when asked.

## Error handling

- **409** — a subscription with that grouping already exists. Read it and edit it with
  `editSubscription` rather than creating a second.
- **400** — a filter references a tag that does not exist for that notification type; re-read
  step 5.
- **403** — notification management is a distinct RBAC permission.

## Note on the deprecated webhook

`registerWebhook` (`POST /cc-ui/v1/onetime-webhook/register`) is marked `deprecated: true` in
the spec. It is a register-then-poll one-time webhook, not part of the subscription model. Do
not use it for new work.
