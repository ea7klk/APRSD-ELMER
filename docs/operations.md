# Operations

## Shape of the deployment

One container running `aprsd server`, one callsign, one configuration file.
No database of its own. Data sources are reached over HTTP inside the cluster;
the geodata and park tables are read-only files in the image.

```
namespace: elmer
├── Deployment  elmer-aprsd        (aprsd server + aprsd-elmer plugin)
├── ConfigMap   elmer-aprsd-conf   (aprsd.conf, non-secret)
├── Secret      elmer-aprsd-secret (aprs_network.password, service tokens)
└── PVC         elmer-data         (optional: aprsd save_location)
```

`aprsd` writes its watch list and packet tracker to `save_location`
(`enable_save`, on by default). That is the only mutable state and it can be an
`emptyDir` if losing it on restart is acceptable — for a stateless answering
service, it is.

## Configuration

Start from [examples/aprsd-elmer.conf.example](../examples/aprsd-elmer.conf.example).
Three settings deserve explanation because their defaults are wrong for this
service.

### Set `enabled_plugins` explicitly

`aprsd` ships a default list that references `aprsd.plugins.location.LocationPlugin`,
which is **not** in the core package — it comes from a separate extension. A
default install will not run the plugin set you expect. Always list plugins by
full dotted path and nothing else.

### Use a European APRS-IS server

`aprs_network.host` defaults to `noam.aprs2.net`. For a station in Spain, use
`euro.aprs2.net`. The login is the base callsign; `aprslib` strips the SSID
before computing the passcode, so `EA7KLK` and `EA7KLK-10` produce the same
passcode, but only the base callsign should be in `aprs_network.login`.

The passcode is not derivable at runtime by design — it is a constant for a
callsign — and `aprsd` fails at startup with `aprs_network.password is not set`
if it is missing. It belongs in the SOPS-encrypted secret, not the ConfigMap.

### Choose the service SSID deliberately

`EA7KLK-10` is used throughout this document as an example. Before deployment,
check that the SSID is free in the region and does not collide with the BBS's
own receiver login, which uses a base callsign or a sysop callsign.

## Build

The image is `python:3.12-slim` plus `libsqlite3-mod-spatialite`, with
`aprsd==4.2.4` and `aprsd-elmer` installed from a pinned wheel.

```dockerfile
FROM python:3.12-slim
RUN apt-get update \
 && apt-get install -y --no-install-recommends libsqlite3-mod-spatialite \
 && rm -rf /var/lib/apt/lists/*
COPY dist/*.whl /tmp/
RUN pip install --no-cache-dir aprsd==4.2.4 /tmp/aprsd_elmer-*.whl
COPY data/ /usr/share/elmer/
ENTRYPOINT ["aprsd", "server", "--config", "/etc/aprsd/aprsd.conf"]
```

`data/` holds `spain_boundaries.sqlite` and the generated `parks.sqlite`.

## First deployment

Do not deploy ELMER first. Deploy stock `aprsd` first, with `PingPlugin` only.

1. `aprsd sample-config` to generate a reference file, then reduce it to the
   options actually used.
2. Run with `enabled_plugins = aprsd.plugins.ping.PingPlugin`.
3. Send `ping` from a radio and confirm the `Pong!` reply arrives, which proves
   login, acknowledgement, retry and reconnect in one step.
4. Kill the container mid-session and confirm it reconnects without duplicate
   replies.
5. Only then add ELMER plugins, one per release.

## Observability

`aprsd dump-stats` reports per-plugin `rx_count`/`tx_count` and the packet
collector's view of the session. Expose it on a sidecar or via `kubectl exec`
for a first look; do not build a metrics endpoint before there is traffic worth
measuring.

The questions that actually get asked in an incident:

- Is the APRS-IS session alive, and how long has it been up?
- Are commands arriving but not answered (plugin exception) or not arriving at
  all (filter or login problem)?
- Is a data source stale, and does the reply say so?

The last one is why every plugin logs its upstream latency and its staleness
decision. A bot that silently degrades is worse than one that is down.

## Runbook

**No replies at all.**
Check the session first (`dump-stats`), then the login response. An
`unverified` or `invalid` login can receive but not send; the BBS treats that as
a failure and so should this station's health check.

**Replies to some callsigns only.**
Filters. A watch-list plugin calls `set_filter('b/call/call')`, which narrows
what the session receives. A command plugin that needs all traffic to its own
callsign must not add a filter that excludes it.

**`collector stale` replies from `?BM`.**
`bminfo`'s BrandMeister collector is behind. That is a `bminfo` incident, not
an ELMER one; the reply exists so the operator does not read it as silence.

**Rate-limit complaints.**
`msg_rate_limit_period` is global per instance, not per sender. If a busy
evening produces queues, the fix is a per-sender bucket in the plugin, not a
lower global rate.

**Callsign complaint.**
ELMER only ever transmits as its configured callsign. If a packet claims
otherwise, it did not come from this service.

## Upgrade policy

Pin `aprsd` to an exact version. The plugin contract (`filter`, `process`,
`create_threads`) is stable but the surrounding threads and config groups are
not; a minor bump should be read before it is applied, and phase 0 exists to
make that cheap to test.
