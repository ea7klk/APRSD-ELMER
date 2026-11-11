# AGENTS.md

This guidance applies to the entire repository.

## Project

This repository holds the design of **ELMER**, an APRS concierge station built
on [`aprsd`](https://github.com/craigerl/aprsd). It is currently a design
proposal: documentation, an example configuration, and no implementation.

Read `README.md` first, then the document relevant to the change.

## Structure

```text
README.md                      the concept, boundaries and roadmap
docs/commands.md               command catalogue and reply formats
docs/data-sources.md           read-only integrations and their constraints
docs/operations.md             deployment, configuration, runbook
docs/risks.md                  regulatory, privacy, abuse and technical risks
examples/aprsd-elmer.conf.example   a real aprsd 4.2.4 configuration, plus [elmer]
```

## Writing conventions

- Write for an operator who has to run this, not for a reader who has to be
  convinced. State the decision, then the reason.
- Prefer plain sentences and short lists. No marketing language.
- Every claim about an external system should name the file, option or model
  it comes from, so it can be re-verified.
- Keep the three concerns separate: what the station answers
  (`commands.md`), what it reads (`data-sources.md`), and what can go wrong
  (`risks.md`).

## Hard constraints to preserve when editing

- **67-character replies.** APRS messages are truncated, not split. Any new
  command format states its measured length and its budget, and the budget
  stays under 67 including an APRS `{msgNo` suffix.
- **ASCII output.** No accented characters reach the reply path.
- **One reply per command.** No command produces more than one packet.
- **Read-only.** No ELMER component writes to another service's data.
- **No user text relay.** The station answers from its own sources and emits
  its own events; it never forwards text between operators.
- **One callsign.** The station transmits only as its configured callsign.

## Relationship to other repositories

ELMER is a separate service identity. It is not part of `simple-ham-bbs`, does
not replace the BBS's per-user APRS layer, and does not depend on the
`bm-messaging` protocol spike. Changes here must not require changes to the
BBS's APRS sending or receiving code.

## Before proposing an implementation

The roadmap in `README.md` starts with stock `aprsd` and no ELMER code. Keep
that ordering: phase 0 exists so the plugin contract, the APRS-IS session and
the deployment path are proven before any custom plugin is written.

## Secrets

Never commit APRS-IS passcodes, service tokens, or callsign-specific
credentials. The example configuration contains placeholders only.
