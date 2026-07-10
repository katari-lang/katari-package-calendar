# calendar — Google Calendar tools for Katari

A single module, `calendar`: two tools the model can call — `list_events` and `create_event` — backed
by Google's OAuth refresh-token grant. Pure Katari over `http.fetch`: the token exchange and both
Calendar API calls build their request and parse their reply as `json`. No FFI sidecar.

- `calendar.list_events(calendar_id, time_min, time_max, max_results?)` — upcoming events in a
  window, trimmed to id / summary / start / end / link.
- `calendar.create_event(calendar_id, summary, start, end, description?)` — create an event.
- `calendar.provider(client_id, client_secret, refresh_token)` — provides the OAuth credentials for
  the extent of a continuation. Each tool call exchanges the refresh token for a fresh access token.
- `calendar.oauth_env(key)` — read a non-secret OAuth env entry (the empty string when unset).

## Secrets / env

Three OAuth values, provisioned as **plain (non-`--secret`) env entries** — this is deliberate:

- `GOOGLE_OAUTH_CLIENT_ID`
- `GOOGLE_OAUTH_CLIENT_SECRET`
- `GOOGLE_OAUTH_REFRESH_TOKEN`

Katari lets a secret leave the runtime only as an auth *header*, but Google's token endpoint reads
these credentials from the `application/x-www-form-urlencoded` request body — a public sink — and a
refresh token has no header form at all. So they cannot be `string of private`; the module reads them
as public strings. Treat them as sensitive at the OS level even though the type system cannot here.

To get a refresh token: create an OAuth client (type "Web application" or "Desktop app") in the
Google Cloud console, enable the Google Calendar API, then use the
[OAuth 2.0 Playground](https://developers.google.com/oauthplayground) with your own credentials to
authorize the `https://www.googleapis.com/auth/calendar` scope and exchange the code — the response's
`refresh_token` is the value to store.

## Usage

```katari
import calendar

agent upcoming(calendar_id: string, time_min: string, time_max: string) -> array[calendar.event] with io {
  use calendar.provider(
    client_id = calendar.oauth_env(key = "GOOGLE_OAUTH_CLIENT_ID"),
    client_secret = calendar.oauth_env(key = "GOOGLE_OAUTH_CLIENT_SECRET"),
    refresh_token = calendar.oauth_env(key = "GOOGLE_OAUTH_REFRESH_TOKEN"),
  )
  calendar.list_events(calendar_id = calendar_id, time_min = time_min, time_max = time_max)
}
```

Hand `calendar.list_events` / `calendar.create_event` to an AI loop's tool list to let the model read
and schedule on its own. A failed call throws `calendar.calendar_error` — handle it at the app root.
