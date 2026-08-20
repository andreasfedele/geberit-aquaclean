# This fork — changes vs. jens62/geberit-aquaclean

Based on [jens62/geberit-aquaclean](https://github.com/jens62/geberit-aquaclean) @ `5cdfdf5`.
Fixes two problems reported upstream:

- [Issue #38](https://github.com/jens62/geberit-aquaclean/issues/38) — `aioesphomeapi` declared without a version constraint
- [Issue #37](https://github.com/jens62/geberit-aquaclean/issues/37) — expose orientation light status and presence as HA entities

## #38 — `aioesphomeapi` version pin

**File:** `pyproject.toml`

The dependency was declared as bare `"aioesphomeapi"` — no version constraint. When
Home Assistant/HACS installs this package's requirements, an unconstrained spec lets
pip pull whatever is newest on PyPI, which can differ from the `aioesphomeapi` build
HA Core's own `esphome` integration already has loaded. Both ship compiled Cython
extensions; two ABI-incompatible copies in one process break *every* ESPHome device
on the installation (RuntimeWarning: "size changed, may indicate binary
incompatibility", missing C symbols, `UnhandledAPIConnectionError`) — exactly what
was reported.

Changed to `aioesphomeapi>=24.6.0,<25.0.0` — centered on 24.6.2, the version this
bridge was last verified against (`docs/cli.md`). This is a mitigation, not a
permanent fix: **before installing on a given HA release, confirm this range still
matches what that release ships** (Settings → System → Repairs → ⋮ on the ESPHome
integration → "Download diagnostics", or `pip show aioesphomeapi` in the HA
venv/container), and adjust the pin in `pyproject.toml` if it doesn't. The bundled
version changes with every HA release; this pin does not update itself.

### Round 2 (`b4`) — the upper bound was itself the bug

`b2`/`b3` widened the range twice (`<25.0.0`, then `<26.0.0` for Python 3.14
wheel availability) but kept an upper bound — and that upper bound caused the
*exact* failure this fix exists to prevent, on a real installation, after a
routine HA restart: every ESPHome device went down with
`ImportError: cannot import name 'VoiceAssistantAnnounceFinished' from
'aioesphomeapi'`.

Root cause, confirmed by reading HA Core's own source for the release in
question (2026.8.2): its `esphome` integration exact-pins
`aioesphomeapi==45.6.1` in its own `manifest.json`, and that exact version
ships pre-baked into the HA Core image — it is **not** reinstalled at
runtime by HA Core itself. The *only* thing on a stock HA install that can
move this shared, globally-installed package at runtime is a
custom_component's own `requirements` entry, which HA re-checks (and
lazily pip-installs to satisfy) on every restart. `b3`'s
`>=25.0.0,<26.0.0` range no longer overlapped with what HA 2026.8.2 actually
needed (45.6.1) — so on every boot, installing/verifying this integration's
requirements silently downgraded the correct, pre-baked 45.6.1 into the
`<26.0.0` range, breaking every ESPHome device until something reinstalled
the right version.

In other words: **any upper bound here is guaranteed to go stale**, because
it's trying to predict a value (HA Core's own exact pin) that only this
fork's maintainer can't control and that moves forward with every HA
release. `b4` drops the ceiling entirely: `aioesphomeapi>=45.6.1`, floor
only, set at the version confirmed needed as of HA 2026.8.2. This
integration only imports the long-stable `APIClient` class (see
`aquaclean_console_app/bluetooth_le/LE/{ESPHomeAPIClient,BluetoothLeConnector}.py`)
— nothing that benefits from capping how high pip is allowed to go — so an
open-ended floor is safe, and it also makes updating to `b4` self-healing:
on the next restart after updating, this integration's own requirements
check will pull the already-broken (too-low) install back up past 45.6.1
to satisfy the new floor, restoring ESPHome without any extra manual step
beyond updating and restarting.

If a future HA release ever needs something below whatever floor is set
here (astronomically unlikely for an actively developed library — version
numbers only go up), bump the floor then. Never add a ceiling again.

### Round 3 (still `b4`) — drop the dependency entirely, adopting upstream PR #46

Before pushing round 2, [upstream PR #46](https://github.com/jens62/geberit-aquaclean/pull/46)
("Avoid overriding Home Assistant ESPHome client", open/unmerged as of this
writing, by `fabpot`) turned up, taking a cleaner approach to the same
issue: move `aioesphomeapi` out of the unconditional base `dependencies`
and into `[project.optional-dependencies] esphome = ["aioesphomeapi"]`.

This fork adopts that approach instead of shipping its own round-2 floor
pin. Reasoning: `aioesphomeapi` is only ever imported for the
ESPHome-Bluetooth-Proxy connection path, and both places that import it
(`aquaclean_console_app/bluetooth_le/LE/BluetoothLeConnector.py` and
`ESPHomeAPIClient.py`) already do so *lazily*, inside functions — not at
module load time. So a plain `pip install geberit-aquaclean` never needed
to install or touch `aioesphomeapi` at all for direct-BLE use; requiring
it unconditionally was the actual mistake behind #38 from the start, not
merely a missing/wrong version constraint. With it moved to an extra:

- A stock HACS/HA install of this fork never installs or upgrades/downgrades
  `aioesphomeapi` — Home Assistant's own `esphome` integration remains the
  sole owner of that shared package, exactly as it should be. There is no
  version to pick, guess, or keep in sync with HA releases anymore, so this
  class of bug cannot recur regardless of what HA Core pins in the future.
- Anyone using `aquaclean_console_app` **standalone** (outside Home
  Assistant) who wants the ESPHome-proxy connection path installs it
  explicitly: `pip install geberit-aquaclean[esphome]`. Pure direct-BLE
  standalone use needs nothing extra, same as before.
- `manifest.json`'s `requirements` entry is unaffected — it still installs
  the base package with no `[esphome]` suffix, so HA installs get exactly
  `bleak`, `aiorun`, `fastapi`, `uvicorn`, `paho-mqtt` and nothing more;
  `aioesphomeapi` comes entirely from HA Core's own `esphome` integration.

If/when PR #46 is merged and released upstream, this fork's fix becomes
redundant with upstream — at that point, switching HACS back to
`jens62/geberit-aquaclean` directly (rather than this fork) is the
simplest long-term path, provided issue #37's entities are also merged
upstream by then, or are otherwise not needed.

## #37 — orientation light status / presence

**Presence** (`is_user_sitting` → "User Sitting", `BinarySensorDeviceClass.OCCUPANCY`)
was **already implemented** in the base branch, for both Mera-family and Alba
devices (`binary_sensor.py`, feature set `FS_ALL`, `wired=True`). No change needed —
if you're on an older release, update the integration to get this entity, no fork
required for that half.

**Orientation light status** was not implemented, and the "obvious" fix — request
`GetSystemParameterList` index 9 (`StateOrientationLight`) — is a trap already
flagged in this codebase's own comments: index 9 is AcSela-only and confirmed
**not valid on Mera Comfort**; requesting it leaves the device in a state where the
next `GetFilterStatus` call times out until it's power-cycled
(`AquaCleanClient.py`, `SPL_PARAMS_MERA_COMFORT`). Wiring that up naively would have
traded a feature request for a real regression.

Added instead: `binary_sensor.orientation_light_on_estimated` ("Orientation Light
(Estimated)"), Mera Comfort only, **disabled by default**, diagnostic category.
It's a derived value, not a direct device readout — computed in
`coordinator._build_mera_result()` from two already-confirmed-working reads:
the orientation-light activation mode (`Off` / `On` / `WhenApproached`, common
setting id 3) and live presence (`is_user_sitting`). Logic: **on** when the mode
is `On`, or when the mode is `WhenApproached` and someone is currently sitting;
**off** otherwise; **unknown** if the mode hasn't been read yet. Enable the entity
manually once you've confirmed on your own unit that it tracks reality — it's
disabled by default specifically because it's inferred, not measured.

## Also bumped

- `pyproject.toml` / `manifest.json` version → `3.1.3b4`
- `manifest.json` `requirements` / `documentation` / `issue_tracker` point at this
  fork's URL — update the org/repo name and the `@v3.1.3b4` tag once you've pushed
  this to your own GitHub, and create a matching git tag, or HACS/pip won't be able
  to resolve the requirement.

## Not verified on real hardware

None of this was tested against a physical Geberit AquaClean — there's no device
attached to this environment. The `aioesphomeapi` pin is a version-string change
only (low risk). The orientation-light entity is new logic built from adjacent
values that are otherwise confirmed working, but the derived on/off result itself
hasn't been checked against a real light state. Verify both before relying on them.
