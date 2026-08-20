# This fork — changes vs. jens62/geberit-aquaclean

Based on [jens62/geberit-aquaclean](https://github.com/jens62/geberit-aquaclean) @ `5cdfdf5`.
Everything is identical to upstream except the fix below for
[Issue #38](https://github.com/jens62/geberit-aquaclean/issues/38).

## #38 — `aioesphomeapi` broke every ESPHome device on the installation

**File:** `pyproject.toml`

`aioesphomeapi` was declared as an unconditional base dependency (originally
with no version constraint at all). Home Assistant's own `esphome`
integration exact-pins its own `aioesphomeapi` version and ships it
pre-baked in the HA Core image — it is not reinstalled at runtime. Any
version constraint this package declares for the same, globally-shared
package name is checked and (re-)installed on every HA restart, which can
silently override HA Core's own correct, pre-baked version — breaking
every ESPHome device on the installation, not just this integration's own
proxy connection. This was confirmed twice: once as originally reported
(unconstrained), and again on a real installation after this fork tried
pinning a version *range* (`>=25.0.0,<26.0.0`), which went stale the moment
HA Core moved to a release needing a newer `aioesphomeapi` than the range
allowed.

**Fix:** `aioesphomeapi` is no longer a base dependency at all — it's moved
to an optional extra:

```toml
[project.optional-dependencies]
esphome = ["aioesphomeapi"]
```

This is the same approach as upstream PR
[jens62/geberit-aquaclean#46](https://github.com/jens62/geberit-aquaclean/pull/46)
("Avoid overriding Home Assistant ESPHome client"), open/unmerged as of
this writing — adopted here rather than waiting on it. It's a correct fix
because `aioesphomeapi` is only ever imported lazily, inside functions, by
the optional ESPHome-Bluetooth-Proxy connection path
(`aquaclean_console_app/bluetooth_le/LE/BluetoothLeConnector.py` and
`ESPHomeAPIClient.py`) — not at module load time, and not by direct-BLE
users at all. So a stock `pip install geberit-aquaclean` (what HA/HACS
does) never touches `aioesphomeapi` on Home Assistant; the integration
simply uses whatever HA Core's own `esphome` component already has
installed, which is always present and correct on any install that also
uses the ESPHome Bluetooth Proxy feature. Standalone (non-HA) console-app
users who want ESPHome-proxy support install the extra explicitly:
`pip install geberit-aquaclean[esphome]`.

## Also changed (fork mechanics, not a behavior change)

- `pyproject.toml` / `manifest.json` `version` bumped so HACS/HA see a new
  release to install.
- `manifest.json` `requirements` / `documentation` / `issue_tracker` point
  at this fork's GitHub URL and release tag instead of upstream's — update
  these (and create a matching git tag) if you re-cut a release from this
  fork.

## If upstream merges PR #46 (or fixes #38 another way)

Switch back to `jens62/geberit-aquaclean` directly and drop this fork —
there's nothing else here to preserve.
