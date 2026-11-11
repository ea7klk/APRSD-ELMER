# Data sources

ELMER is a thin answering layer. Everything it knows comes from a service in
this estate, read-only. No plugin in this design owns a database or writes to
one.

## BrandMeister last-heard — `bminfo`

Repository: [`ea7klk/bminfo`](https://github.com/ea7klk/bminfo)
Store: PostgreSQL, namespace `bminfo`

Relevant models (verified in `bminfo/dashboard/models.py`):

- `QSO` — `source_call`, `destination_call`, `context_id` (talkgroup),
  `start_at`, `stop_at`, `master`, `slot`, `duration_ms`.
- `Talkgroup` — `talkgroup_id`, `name`, `country`.
- `ServiceHeartbeat` — `service_name`, `last_seen_at`.

The `?BM` query is one indexed lookup: most recent `QSO` for `source_call`
within the lookback window. The `Talkgroup` join exists only to resolve an id
to a name when it fits the budget; it usually does not, so the numeric id is
the default.

**Staleness is a first-class answer.** If `ServiceHeartbeat` for the collector
is older than a configured threshold, `?BM` replies `collector stale <n>m`
rather than `not heard`. A silent collector and a quiet operator must not look
the same.

Access options, in order of preference:

1. **A read-only HTTP endpoint on `bminfo`.** One view, one serializer, a
   service token. Keeps PostgreSQL and its credentials out of the ELMER
   container entirely, and gives the endpoint a place to enforce the field
   allowlist.
2. **A read-only PostgreSQL role** plus a NetworkPolicy permitting only the
   ELMER namespace. Acceptable, but it couples two deployments' lifecycle.

Not acceptable: sharing the application's database user, or letting ELMER read
`User`, `EmailVerificationToken`, or any account table.

## Places — `SQLite-Geo`

Repository: [`ea7klk/SQLite-Geo`](https://github.com/ea7klk/SQLite-Geo)
Store: `spain_boundaries.sqlite`, SpatiaLite

The dataset carries countries, ADM1/ADM2 subdivisions, Spanish municipalities,
traditional comarcas, and judicial districts. It requires the
`mod_spatialite` extension (`libsqlite3-mod-spatialite` on Debian), which is
the one non-Python dependency of the whole service.

ELMER needs exactly one query — point in polygon for municipality, province and
comarca — and it must run in a few hundred milliseconds, because an APRS reply
that takes five seconds has already been forgotten. The dataset is read-only
and can be baked into the image or mounted from a PVC; either way it is
refreshed by `SQLite-Geo`'s own updater, not by ELMER.

If the spatial extension is unavailable, the plugin degrades to nearest-centroid
matching against a much smaller precomputed table and says so in its log, not
in the reply.

## POTA parks

Sources already in the estate:

- `pota_filtered.osm.pbf` — a filtered OSM extract (~50 MB) of POTA-tagged
  objects.
- [`ea7klk/overpass-radio-replication-server`](https://github.com/ea7klk/overpass-radio-replication-server)
  — a CronJob every four hours that maintains an Overpass database of
  `communication:amateur_radio*` objects.
- [`ea7klk/OSM-POTA-Map`](https://github.com/ea7klk/OSM-POTA-Map) — the
  rendered park map.

**Do not query Overpass at request time.** An overpass query in the reply path
is a multi-second dependency on a third-party service and a shared public API.
Instead, a build step extracts park references, names and centroid points from
the PBF into a small SQLite table (`parks.sqlite`), which ships with the image
and is refreshed on the same cadence as the extract. `?NEAR` then does a
haversine scan over a few thousand rows, which is microseconds.

Park names are also transliterated to ASCII at build time, so the reply path
does no Unicode work at all.

## Contest windows — `contestmaster`

Repository: [`ea7klk/contestmaster`](https://github.com/ea7klk/contestmaster)
API: FastAPI, public routers under `backend/app/api/`

`Contest` (`backend/app/models/domain.py`) carries `name`, `slug`, `starts_at`,
`ends_at`, `status`, and `log_submission_cutoff_at`. The public contest router
already exposes published contests without authentication.

`?TEST` needs one field that may not be public yet: a list of contests starting
in the next N days, sorted. If the existing public API cannot express that
without authentication, the right fix is a small public endpoint
(`/api/v1/contests/upcoming`) in Contestmaster, not a database credential in
ELMER.

## BBS mail — `simple-ham-bbs` / `ham-bbs-web`

Repository: [`ea7klk/simple-ham-bbs`](https://github.com/ea7klk/simple-ham-bbs),
[`ea7klk/ham-bbs-web`](https://github.com/ea7klk/ham-bbs-web)
Store: `bbs.sqlite`, shared by both surfaces

Relevant models: `dbUser`, `dbBoard`, `dbMessage`, `dbBulletin`. The APRS
tables (`dbAPRSSent`, `dbAPRSSentPart`, `dbAPRSReceived`) belong to the BBS's
own APRS layer and are **not** read by ELMER.

`?MAIL` goes through a token-protected endpoint on `ham-bbs-web`, as described
in [commands.md](commands.md#v2). The rationale is the same in both directions:
the web app already owns the locking and sanitisation rules for that file, and
a second reader in a different language is how schema drift becomes a
production incident.

## What ELMER deliberately does not read

- BBS passwords, session tokens, email addresses, or message bodies.
- `bminfo` account tables.
- APRS-IS passcodes for anyone but its own callsign.
- Anything that would let one operator learn another's private traffic.

## Freshness summary

| Source | Refresh | Failure answer |
|---|---|---|
| BrandMeister QSOs | live stream via collector | `collector stale` |
| Places | manual updater | nearest-centroid fallback |
| Parks | build step, with extract | omit park, still answer place |
| Contests | live API | `contestmaster unavailable` |
| BBS mail | live endpoint | `mail unavailable` |
