# Command catalogue

Every command is a single APRS message addressed to the service callsign
(`EA7KLK-10`), begins with `?`, and is answered with one line of at most 67
ASCII characters.

Anything that does not match a command is ignored. `aprsd`'s built-in
`HelpPlugin` answers `h`/`help`, and remains enabled
(`load_help_plugin = true`).

## v1

### `?BM <CALLSIGN>`

BrandMeister last-heard.

```
?BM EA7ABC
-> EA7ABC TG214 14:32 BrandMeister_ES
```
34 characters measured; the format budget is 40.

Reply shape — `<call> TG<id> <HH:MM UTC> <master>`:

- `TG<id>` is the talkgroup (`QSO.context_id`, resolved to `Talkgroup.name`
  only when the numeric id would not fit).
- The timestamp is UTC with no date, because a message from last week is not
  "last heard" in any useful sense.
- `master` is the server name, truncated to its first token
  (`BrandMeister_ES` from `BrandMeister_ES_...`).

Failure shapes, all distinct on purpose:

```
?BM EA7ABC  -> EA7ABC not heard 24h
?BM EA7ABC  -> ?BM unknown call
?BM          -> ?BM needs a call
?BM EA7ABC  -> ?BM collector stale 42m
```

The last one matters: if `bminfo`'s collector heartbeat is old, "not heard" is
a lie. The plugin checks `ServiceHeartbeat` before it answers, and says so.

Replies never include names, locations, or IP-adjacent detail — only what the
public LastHeard pages already show.

### `?NEAR`

Where am I? Reverse geocode the sender's last known position, then the nearest
POTA park within a radius.

```
?NEAR
-> EA-0391 Despenaperros 12km N | Jaen
```
35 characters measured; the format budget is 55.

- Position comes from the sender's most recent APRS position packet, as seen
  by `aprsd`'s own packet tracker (or an aprs.fi lookup if it is stale).
- Municipality and province come from the local SpatiaLite dataset, so the
  answer is available with no external geocoding service.
- The park reference and name come from a precomputed park table (see
  [data-sources.md](data-sources.md#pota-parks)).
- Distance and bearing are truncated, not rounded up: `12km N`, never
  `~12.4 km north-north-east`.

Failure shapes:

```
?NEAR  -> no position for EA7ABC
?NEAR  -> Malaga, Andalucia | no park 25km
```

Note the second: absence of a park is still a useful answer, so it is not an
error.

### `?TEST`

Next contest window from Contestmaster.

```
?TEST
-> CQWWCW starts 27/11 00:00Z in 15d
```
33 characters measured; the format budget is 45.

- Reads `Contest.starts_at` / `ends_at` for contests that are public and not
  drafts.
- Long names are abbreviated to a callsign-style token; the point is
  recognition, not fidelity.
- Within 24 hours the reply switches to hours (`in 6h`), because "1d" hides
  the fact that it starts tonight.

```
?TEST  -> no contest in 30d
?TEST  -> contestmaster unavailable
```

## v2

### `?MAIL <CALLSIGN>`

Unread BBS replies, count and most recent subject only.

```
?MAIL EA7ABC
-> 3 new, 1 unread: Re: WG mesh
```
28 characters measured; the format budget is 50.

Deliberately not implemented against the BBS database file. Reading
`bbs.sqlite` from a second container means a third writer on the same SQLite
file and a schema coupling between a Python plugin and Go-owned tables.
Instead, `ham-bbs-web` — which already reads the same database with the right
locking discipline — exposes a token-protected read endpoint:

```
GET /api/elmer/mail?callsign=EA7ABC
-> {"new": 3, "unread": 1, "last_subject": "Re: WG mesh"}
```

The endpoint is scoped to a service token, returns no message bodies, and
answers 404 for unknown callsigns. ELMER never sees a password, an email
address, or message text.

## Push messages

Push is not a command. It is a worker that emits service events to an opt-in
allowlist.

```
EA7ABC on BM TG214 after 9h quiet
```
33 characters measured, and it is sent **once** per crossing — not once per transmission.

Rules:

- Subscription is per callsign and per event type, stored in configuration, not
  in a database the bot can write.
- A watched callsign can be added by the operator who wants it; there is no
  self-service path in v1.
- Minimum re-notify interval per watched callsign, independent of `aprsd`'s
  own duplicate window.
- The bot never reports where a station is, only that it was heard, and only to
  the subscriber.

## Reply formats at a glance

| Command | Pattern | Max observed length |
|---|---|---|
| `?BM` | `<call> TG<id> <HH:MM> <master>` | 40 |
| `?NEAR` | `<ref> <park> <dd>km <bearing> \| <municipality>` | 55 |
| `?TEST` | `<token> starts <dd/mm> <HH:MM>Z in <n>d` | 45 |
| `?MAIL` | `<n> new, <n> unread: <subject>` | 50 |

All four have headroom under 67, which is where the `{msgNo` suffix and any
future prefix have to live.
