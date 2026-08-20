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

- `pyproject.toml` / `manifest.json` version → `3.1.3b2`
- `manifest.json` `requirements` / `documentation` / `issue_tracker` point at this
  fork's URL — update the org/repo name and the `@v3.1.3b2` tag once you've pushed
  this to your own GitHub, and create a matching git tag, or HACS/pip won't be able
  to resolve the requirement.

## Not verified on real hardware

None of this was tested against a physical Geberit AquaClean — there's no device
attached to this environment. The `aioesphomeapi` pin is a version-string change
only (low risk). The orientation-light entity is new logic built from adjacent
values that are otherwise confirmed working, but the derived on/off result itself
hasn't been checked against a real light state. Verify both before relying on them.
