# ELMER — an APRS concierge station

**Status: design proposal.** Nothing described here is deployed, and no
interface in this document is stable. The project may be rewritten or dropped.

ELMER is an amateur radio station with a callsign and no radio: an
[`aprsd`](https://github.com/craigerl/aprsd) instance that answers one-line
questions sent over APRS and pushes the few events worth interrupting an
operator for.

It exists because three services in this estate already know things no public
APRS bot knows — who is on BrandMeister right now, which contest is about to
start, and which park or municipality you are sitting in — and none of them are
reachable from a radio.

```
   any APRS radio  ──▶  APRS-IS  ──▶  EA7KLK-10 (aprsd + aprsd-elmer)
                                          │
                        ┌─────────────────┼─────────────────┐
                        ▼                 ▼                 ▼
                   bminfo lastheard   contestmaster    SQLite-Geo / POTA
                   (BrandMeister)     (contests)       (places, parks)
```

## Why a separate station, not part of the BBS

[`simple-ham-bbs`](https://github.com/ea7klk/simple-ham-bbs) already sends and
receives APRS, but it does so **per user**: the source address is whichever
callsign is logged in, long bodies are split into numbered parts with per-part
message IDs, and inbound acknowledgements are matched back to those parts
against the BBS database.

ELMER is the opposite shape: **one callsign answering everyone**. That is a
different identity model, a different failure mode, and a different trust
boundary. It runs as its own process with its own callsign, and it never speaks
for a user.

| | SSH BBS | ELMER |
|---|---|---|
| APRS source | the logged-in user's callsign | one service callsign (`EA7KLK-10`) |
| Carries user text | yes, message bodies | no, only service-originated data |
| State | BBS SQLite | stateless; reads other services |
| Transport | one socket per send | one long-lived APRS-IS session |

The "carries user text" row is the important one. ELMER never relays a message
between two people, which is why it can be operated unattended without
entering third-party traffic territory.

## What an operator sees

A licensed operator sends an APRS message to `EA7KLK-10`:

```
?BM EA7ABC      ->  EA7ABC TG214 14:32 BrandMeister_ES
?NEAR           ->  EA-0391 Despenaperros 12km N | Jaen
?TEST           ->  CQWWCW starts 27/11 00:00Z in 15d
?MAIL EA7ABC    ->  3 new, 1 unread: Re: WG mesh
```

Every reply is designed against APRS's hard 67-character ceiling (see
[The 67-character contract](#the-67-character-contract)) and every command is
read-only.

## Why aprsd

`aprsd` 4.2.4 (Apache-2.0, Python ≥ 3.10) does the parts that are tedious and
easy to get wrong:

- one long-lived APRS-IS session with reconnect and backoff, or a KISS TNC if
  the station ever moves to RF;
- automatic acknowledgements with a send retry count;
- duplicate suppression across reconnects (`packet_dupe_timeout`, 300 s);
- outbound rate limiting per packet class (acks 1/s, messages 1 per 2 s by
  default), which is exactly the guard a public bot needs;
- a plugin contract, so ELMER ships as its own package from PyPI and the
  upstream project stays unforked.

It does not provide the answers. Those come from this estate's services.

## The plugin package

```
aprsd_elmer/
├── plugins/
│   ├── near.py      ?NEAR  — reverse geocode + nearest park
│   ├── bm.py        ?BM    — BrandMeister last-heard
│   ├── test.py      ?TEST  — upcoming contest window
│   └── mail.py      ?MAIL  — unread BBS replies (v2)
├── sources/
│   ├── geodata.py   read-only SpatiaLite queries
│   ├── bminfo.py    read-only last-heard lookup
│   └── contest.py   contestmaster public API client
└── workers/
    └── presence.py  watch-list push worker
```

Request/response plugins subclass `APRSDRegexCommandPluginBase`: set
`command_regex`, `command_name`, `short_description`, and implement `process()`.
Push work subclasses `APRSDPluginBase` and returns threads from
`create_threads()`, which `aprsd` starts when the plugin is enabled.

See [docs/commands.md](docs/commands.md) for the command catalogue and reply
formats, [docs/data-sources.md](docs/data-sources.md) for the integrations, and
[docs/operations.md](docs/operations.md) for deployment.

## The 67-character contract

APRS messages are short, and `aprsd` **truncates rather than splits**:
`MessagePacket._filter_for_send` applies `msg[:67]` before sending, and the
source comment still reads "feature req: break long ones into two msgs". The
`{msgNo` suffix APRS appends is counted inside that budget.

Consequences for every plugin here:

1. A reply is a fixed-shape one-liner, never a sentence.
2. Reply text is ASCII. Spanish place names lose their accents
   (`Despeñaperros` → `Despenaperros`) because APRS carries 7-bit ASCII;
   translation happens in the plugin, not in the data source.
3. Missing data gets a short, explicit answer (`?BM no data 6d`), not a
   fallback that looks like success.
4. Anything that cannot fit is a *pointer*, not a payload.

## Rate, budget and abuse

A public APRS bot is a shared resource on a shared network. The limits are part
of the design, not tuning:

- `msg_rate_limit_period = 2` and `ack_rate_limit_period = 1` are `aprsd`
  defaults and stay.
- On top of that, each plugin applies a per-sender token bucket, so one station
  cannot monopolise the callsign.
- Replies are only sent to the sender of a command. There is no broadcast path
  except the opt-in push worker.
- No command writes anything, anywhere. `?MAIL` reads a count and a subject,
  never a body.
- Push targets are an explicit allowlist. A callsign is only watched if the
  operator who added it can name it.

## Risks

Documented in full in [docs/risks.md](docs/risks.md). The short version:

- **APRS-IS source address honesty.** ELMER transmits as its own callsign and
  nothing else. Any design that lets a caller choose the source is off the
  table.
- **Presence aggregation.** BrandMeister last-heard is public one lookup at a
  time; turning it into a subscription feed is a different act. Opt-in,
  per-callsign, with an opt-out list.
- **Unattended RF.** APRS-IS only in v1. A KISS TNC makes this an unattended
  automatic station with identification and third-party obligations, and that
  needs its own review before it happens.
- **Data staleness.** A silent collector read as "not heard" is worse than an
  error. `ServiceHeartbeat` is checked on every `?BM` query.
- **BrandMeister SMS.** [`bm-messaging`](https://github.com/ea7klk/bm-messaging)
  is a receive-only protocol spike, not a product. Nothing here depends on it.

## Roadmap

- **Phase 0 — stock `aprsd`.** Deploy upstream with `PingPlugin` only. Prove
  login, ack, retry, reconnect, and the GitOps path. No ELMER code.
- **Phase 1 — `?BM` and `?NEAR`.** Read-only, no user data, no BBS coupling.
  The first version worth telling anyone about.
- **Phase 2 — `?TEST`.** Contestmaster public API.
- **Phase 3 — `?MAIL` and push.** Requires a token-protected read endpoint on
  `ham-bbs-web` rather than a second reader of the BBS database.
- **Phase 4 — RF.** KISS TNC on a local digipeater, after an unattended-station
  review.

## Non-goals

- Not a replacement for the APRS layer in `simple-ham-bbs`. The BBS keeps
  per-user sending, multipart bodies, and ACK matching.
- Not a message relay. ELMER answers questions and emits its own events; it
  does not carry traffic between two operators.
- Not a DMR or SMS gateway. The Rewind protocol work in `bm-messaging` is not
  part of this service.
- Not an RF station in v1.

## Licence

Not yet chosen. Nothing here is distributed for reuse until that is settled.
