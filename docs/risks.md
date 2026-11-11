# Risks and constraints

## Regulatory

### Third-party traffic

ELMER must never relay text between two operators. It answers a question from
its own data and it emits its own events. The moment it forwards a message
body, it is carrying third-party traffic, and the rules for that differ by
country.

This is why `?MAIL` returns a count and a subject, never a body, and why there
is no "send a message to another callsign" command. If a user wants message
relay, they have the BBS, where accounts, identity and accountability exist.

### Unattended automatic station

APRS-IS only in v1. A KISS TNC turns this into an unattended automatic station
on RF, with obligations around identification, control operator, and
third-party traffic that a TCP session does not have. Phase 4 requires its own
review; it is not a configuration change.

### Identification

The station transmits only as its configured callsign. Every packet's source is
set from configuration, never from the caller, never from the message text.

## Privacy and social

### Presence aggregation

BrandMeister LastHeard is public, one lookup at a time. A subscription that
says "EA7ABC just came on the air" is a different act: it converts an
on-demand public record into continuous monitoring of a person's activity.

Mitigations, all required:

- The subscriber names the callsign explicitly; there is no discovery, no
  "watch everyone in my province".
- Opt-out list checked before any push, maintained by the operator.
- No location data in presence notifications, only the fact of being heard.
- Notifications go to the subscriber only, never to a group or a channel.

### Data minimisation

No plugin reads an email address, a password hash, a session token, or a
message body. This is enforced by the endpoints ELMER is allowed to call, not
by good intentions in a plugin.

## Abuse

### Sender-level flooding

`aprsd`'s global rate limits protect the network from the *service*. They do
not protect the service from *one sender*. Each plugin therefore applies a
per-sender token bucket, and a sender that trips it gets silence, not an error
message — an error message is itself a transmission and doubles the load.

### Answer amplification

No command may produce more than one reply packet. No command may accept a
list, a range, or a wildcard. Inputs are one callsign or nothing.

### Spoofed senders

APRS has no authentication. A packet can claim any source. ELMER must not
treat a source callsign as an authorisation — which is why nothing it does has
a privileged variant, and why push subscriptions live in configuration rather
than being settable over the air.

## Technical

### Truncation, not splitting

`MessagePacket._filter_for_send` applies `msg[:67]`. A long reply is silently
cut, not split. Every reply format is designed to a budget and the budget is
asserted in tests, not reviewed by eye.

### ASCII only

APRS carries 7-bit ASCII. Spanish and Basque place names lose their accents and
`ñ` becomes `n`. Transliteration happens in the data adapter, so the reply path
never produces a character that the network will mangle.

### Filters are sticky

`set_filter` narrows what the session receives for as long as it is connected.
A watch-list plugin that sets `b/call1/call2` during setup silently starves
every command plugin that needs traffic addressed to the service callsign.
Filters are additive in effect and global in scope; this has to be tested with
both plugin families enabled.

### Duplicate suppression is time-based

`packet_dupe_timeout` is 300 s. On RF, the same packet arriving twice from two
digipeaters is normal; on APRS-IS it usually is not. A push worker needs its
own longer per-callsign cooldown, because "seen again five minutes later" is
exactly the case a user does not want to be told about.

### Remote dependency latency

An APRS reply that takes five seconds reads as a dead station. Every source has
a hard timeout well below one second and a defined answer for exceeding it.
Overpass in the request path is forbidden for this reason; parks are
precomputed.

## Dependency and lifecycle

### `aprsd` version drift

The plugin contract is small and stable; the config groups and thread model are
not. Pin the version, and keep phase 0 (stock upstream) deployable so a bump
can be evaluated without ELMER in the picture.

### `bm-messaging` is not a dependency

The BrandMeister SMS/Rewind work is a receive-only spike. Nothing in ELMER
depends on it, and the design should stay that way until that project becomes
a supported service.

### Callsign collisions

The BBS's APRS receiver logs in with a base callsign or a sysop callsign and
runs `filter t/m`. ELMER needs its own SSID and its own login. Two services
sharing one APRS-IS login will fight over the session, and the symptom is
intermittent, which is the worst kind.
