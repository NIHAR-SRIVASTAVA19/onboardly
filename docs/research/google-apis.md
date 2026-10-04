# Google APIs: Gmail send and Calendar freebusy / events / Meet

Resolves issue #11. Primary sources only (Google developer and Cloud help docs), fetched 2026-10-04. Items the docs did not state are flagged **Not confirmed**.

## Summary

- Gmail send needs `gmail.send` (sensitive). Calendar needs `calendar.events` for event + attendees + Meet, and `calendar.freebusy` for availability.
- Meet link is created inside `events.insert` via `conferenceData.createRequest` with `conferenceDataVersion=1`. No separate Meet API or scope.
- Quotas are generous relative to onboarding volume; not a design constraint.
- Testing-mode consent screen expires refresh tokens after 7 days. Production + sensitive scopes needs verification, and unverified apps are capped at 100 new users.
- Shared-mailbox OAuth (one consented user, stored refresh token) works on consumer Gmail and Workspace. Service-account domain-wide delegation is Workspace-only and needs a super admin.

## Scopes

| Need | Scope | Source |
|---|---|---|
| Send mail | `https://www.googleapis.com/auth/gmail.send` ("Send email on your behalf"), classified **Sensitive** | [Gmail scopes](https://developers.google.com/workspace/gmail/api/auth/scopes) |
| Free/busy query | Any of `calendar.freebusy`, `calendar.events.freebusy`, `calendar.readonly`, `calendar` (docs mark authorization "optional" for this endpoint) | [freebusy.query](https://developers.google.com/workspace/calendar/api/v3/reference/freebusy/query) |
| Create event with attendees + Meet | Any of `calendar.events`, `calendar.events.owned`, `calendar.app.created`, `calendar` | [events.insert](https://developers.google.com/workspace/calendar/api/v3/reference/events/insert) |

Notes:
- The create-events guide states the scope `https://www.googleapis.com/auth/calendar` for event creation ([guide](https://developers.google.com/workspace/calendar/api/guides/create-events)), but the reference lists narrower ones. Narrowest practical set: `gmail.send`, `calendar.freebusy`, `calendar.events`. Google says to choose the most narrowly focused scope ([Calendar scopes](https://developers.google.com/workspace/calendar/api/auth)).
- `gmail.compose` is **Restricted** (drafts + send); avoid it. `gmail.send` is the narrower sensitive option.
- **Not confirmed:** the Calendar scopes page does not state a sensitivity class for `calendar.events` / `calendar.freebusy`. Check the scope list shown in the Cloud Console consent-screen scope picker (it categorizes scopes automatically, per [verification overview](https://support.google.com/cloud/answer/13463073)).
- `calendar.app.created` and `calendar.events.owned` exist as narrower alternatives for event creation; whether they fit was not evaluated.

## Creating an event with attendees and a Meet link

Per the [create-events guide](https://developers.google.com/workspace/calendar/api/guides/create-events) and [events.insert](https://developers.google.com/workspace/calendar/api/v3/reference/events/insert):

- `attendees`: array of `{email}`.
- `sendUpdates`: `all` (notify everyone), `externalOnly`, or `none`. `sendNotifications` is deprecated.
- Meet: set `conferenceData.createRequest.requestId` (unique string) and pass query param `conferenceDataVersion=1` (default 0 ignores conference data). Conference type is `hangoutsMeet`.
- Creation is asynchronous: `conferenceData` status is `pending`, then `success`. Read the event back to get the Meet URL if not populated in the insert response.

```python
service.events().insert(
    calendarId="primary",
    body={"summary": "...", "start": {...}, "end": {...},
          "attendees": [{"email": "a@example.com"}],
          "conferenceData": {"createRequest": {"requestId": "<uuid4>"}}},
    conferenceDataVersion=1,
    sendUpdates="all",
).execute()
```

Free/busy: `freebusy.query` accepts at most 50 calendars (`calendarExpansionMax`) and 100 calendar ids per group (`groupExpansionMax`); exceeding returns an error, not partial results ([reference](https://developers.google.com/workspace/calendar/api/v3/reference/freebusy/query)).

## Quotas and rate limits

Gmail ([usage limits](https://developers.google.com/workspace/gmail/api/reference/quota)):
- 1,200,000 quota units/min per project; 6,000 units/min per user per project.
- `messages.send` costs 100 units, so about 60 sends/min per user.
- 500 recipients per message.
- Daily threshold 80,000,000 units per project (billing threshold).
- **Not confirmed:** per-day sending caps for the mailbox itself (Gmail/Workspace product limits) are not on the API docs.

Calendar ([quota guide](https://developers.google.com/workspace/calendar/api/guides/quota)):
- 10,000 requests/min per project; 600/min per user per project; 1,000,000/day billing threshold.
- Exceeding returns `403` or `429` `usageLimits`. Bursts over the per-minute quota cause limiting in the next window.
- Recommended handling: truncated exponential backoff, `min((2^n) + random_ms, max_backoff)`, jitter up to 1,000 ms, max backoff 32-64 s.
- With domain-wide delegation, set `quotaUser` to attribute usage per user.
- **Not confirmed:** per-event attendee limits and event-creation rate limits are not on the fetched API pages.

## OAuth consent screen: Testing mode and refresh tokens

- External app with publishing status **Testing**: refresh token expires in 7 days, unless the only scopes are a subset of name, email, profile ([OAuth 2.0 overview](https://developers.google.com/identity/protocols/oauth2)). Gmail/Calendar scopes are not in that exception, so an unattended sender breaks weekly.
- Testing mode is limited to 100 test users ([manage audience](https://support.google.com/cloud/answer/15549945)).
- Other refresh-token invalidation causes ([same overview](https://developers.google.com/identity/protocols/oauth2)): user revokes access; unused for 6 months; password change (for tokens with Gmail scopes); more than 100 refresh tokens per account per client ID (oldest silently invalidated); admin restricts the services.
- Implication: a password change invalidates the stored token for the shared mailbox. Plan for re-consent and detection of `invalid_grant`.

## Verification at small scale

- Apps requesting sensitive or restricted scopes must complete OAuth app verification ([verification overview](https://support.google.com/cloud/answer/13463073), [requirements](https://support.google.com/cloud/answer/13464321)). Requirements include brand verification, a demo video, scope justification; restricted scopes also need an annual third-party security assessment.
- Unverified apps in production show the unverified-app warning and are capped at 100 new users total over the project lifetime, not resettable ([manage audience](https://support.google.com/cloud/answer/15549945)).
- Production requirements include verified domains, HTTPS redirect URIs, public homepage, privacy policy, terms ([policy compliance](https://developers.google.com/identity/protocols/oauth2/production-readiness/policy-compliance)).
- Internal user type limits authorization to members of the Cloud organization (Workspace only) ([manage audience](https://support.google.com/cloud/answer/15549945)).
- Consequence for a small single-sender app with `gmail.send` (sensitive): (a) Testing mode with 7-day refresh tokens, (b) In production unverified, within the 100-user cap and with a warning screen, (c) verification, or (d) Workspace Internal app. **Not confirmed:** the fetched pages did not describe a "personal use / own account" verification exemption; the "When is verification not needed" section was referenced but not retrieved.

## Building MIME messages from Python

From the [sending guide](https://developers.google.com/workspace/gmail/api/guides/sending):
- Build with `email.message.EmailMessage`, base64url-encode `message.as_bytes()`, send as `{"raw": ...}` to `users().messages().send(userId="me", ...)`.
- Threading replies needs matching `Subject` and RFC 2822 `References` / `In-Reply-To`.
- **Not confirmed:** attachment size limit is not given in that guide.

```python
import base64
from email.message import EmailMessage

msg = EmailMessage()
msg["To"], msg["From"], msg["Subject"] = to, sender, subject
msg.set_content(text)
raw = base64.urlsafe_b64encode(msg.as_bytes()).decode()
service.users().messages().send(userId="me", body={"raw": raw}).execute()
```

## Shared-mailbox OAuth vs service account with domain-wide delegation

| | Shared-mailbox OAuth (user consent) | Service account + DWD |
|---|---|---|
| Account types | Consumer Gmail or Workspace | **Workspace only**; consumer Gmail cannot use DWD |
| Setup | One-time interactive consent as the mailbox user; store refresh token | Super admin adds the service account's numeric Client ID and scopes in Admin console (Security > API Controls); up to 24 h to propagate |
| Identity | The consenting user | Service account impersonates a user via `credentials.with_subject("user@example.org")`; access limited by that user's permissions and the granted scopes |
| Token lifecycle | Subject to Testing 7-day expiry, password-change and revocation invalidation | No user refresh token; credential is a service-account key |
| Verification | Sensitive-scope verification/cap applies for external apps | Admin grant replaces user consent (inference; the docs do not state this explicitly) |

Source: [service account guide](https://developers.google.com/identity/protocols/oauth2/service-account).

## Open items for the map owner

1. Confirm the Calendar scope sensitivity classes in the Cloud Console scope picker.
2. Decide Testing vs production-unverified vs Workspace Internal; this determines user OAuth vs DWD. Whether the sending account is Workspace or consumer Gmail is the deciding input.
3. Gmail mailbox-level daily send limits are outside the API docs.
