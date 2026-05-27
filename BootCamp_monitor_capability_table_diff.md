# BootCamp Monitor Capability Table Diff

Scope: compare the static monitor capability override tables in the local BootCamp AMD drivers:

- Old / 4K fallback driver: `tmp_bc_amd_compare/4k_atikmdag.sys`
- New / 5K-capable driver: `tmp_bc_amd_compare/5k_atikmdag.sys`

Extracted CSV artifacts:

- `tmp_bc_amd_compare/monitor_cap_4k_6136.csv`
- `tmp_bc_amd_compare/monitor_cap_5k_6237.csv`
- `tmp_bc_amd_compare/monitor_cap_diff_6136_vs_6237.csv`

IDA anchors:

- Old driver:
  - `0xae1c08` -> `MonitorCapabilityTableStore_ctor`
  - `0x1405db0` -> `MonitorCapabilityOverrideTable_109Entries`
  - `0x1405d60` -> `MonitorCapabilityOverrideTable_5Entries`
- New driver:
  - `0xaf9540` -> `MonitorCapabilityTableStore_ctor`
  - `0x1416db0` -> `MonitorCapabilityOverrideTable_118Entries`
  - `0x1416d60` -> `MonitorCapabilityOverrideTable_5Entries`

## Row Format

Each static table row is 16 bytes:

```text
u32 manufacturer_id
u32 product_id
u32 raw_capability_code
u32 value
```

The constructor also supports dynamic rows from config:

- `DALPanelPatchByID` via `LoadDalPanelPatchByIdEntries`
- `DALMonitorStereoSupport` via `LoadDalMonitorStereoSupportOverride`

Those dynamic paths use the same row format, but the APP rows below are static rows in the new 118-entry table.

## Count Delta

- Old main table: 109 rows
- New main table: 118 rows
- Net growth: +9 rows

The first 105 rows are stable by row identity. The tail changed.

## New-Only Rows

```csv
index,vendor,product,raw_capability,value
105,0xf022,0x317b,0x32,0x0
106,0xc315,0x1827,0x33,0x0
108,0x7204,0x0243,0x35,0x0
109,0x7204,0x01ec,0x35,0x0
110,0x1006,0xae01,0x36,0x0
111,0x1006,0xae02,0x36,0x0
112,0x1006,0xae0d,0x36,0x0
113,0x1006,0xae0e,0x36,0x0
114,0x1006,0xae05,0x36,0x0
115,0x1006,0xae06,0x36,0x0
116,0x1006,0xae09,0x36,0x0
117,0x1006,0xae0a,0x36,0x0
```

## Old-Only Rows

```csv
index,vendor,product,raw_capability,value
105,0xc315,0x1827,0x32,0x0
107,0x7204,0x0243,0x34,0x0
108,0x7204,0x01ec,0x34,0x0
```

## Meaningful 5K Finding

The new driver adds eight APP rows with raw capability `0x36`:

```text
APP/AE01, APP/AE02, APP/AE05, APP/AE06,
APP/AE09, APP/AE0A, APP/AE0D, APP/AE0E
```

Earlier RE mapped this path:

```text
raw capability 0x36
  -> TranslateMonitorCapabilityCodeToBitIndex() returns internal bit 56
  -> MonitorCaps_SetCapabilityBit() sets capability_mask[6] |= 0x02
  -> DisplayCapabilityService_RefreshCapsAndMaybeForceEdid()
  -> force=1 argument to DP-sink EDID refresh
  -> DalEdidRefreshIfNeededOrForced()
```

So the extracted table diff confirms that BootCamp 6.0.6237 added panel-ID-gated APP quirks for the new forced-EDID runtime path.

## Caution

This table does not contain APP/AE25 or APP/AE26 raw-`0x36` rows. Our iMac19,1 logs involve later panel IDs such as AE25/AE26, so this old BootCamp delta proves the mechanism but not the exact iMac19,1 product table.

Linux takeaway: model the mechanism, not the literal old IDs. Gate any forced EDID/tile recovery on a narrow Apple internal tiled-panel identity/topology.

## Follow-Up Extraction Pass

The old-vs-new comparison was extended beyond the static table.

Useful negative results:

- Both drivers already contain SLS/topology/tiled-display strings and the trace/test string `Test1 5k SetMode`.
- Both drivers already have DisplayID parser support for tag `0x70` and DisplayID `0x11..0x13` style blocks.
- Both drivers have matching DisplayID sub-block parse/cache logic and timing/mode extraction logic.
- The post-EDID mode/capability rebuild path exists in both drivers.

The meaningful runtime delta is therefore narrower:

```text
old 6.0.6136:
  no APP raw-0x36 rows
  TranslateMonitorCapabilityCodeToBitIndex(0x36) returns 0
  EDID refresh path has no force argument

new 6.0.6237:
  APP raw-0x36 rows for AE01/AE02/AE05/AE06/AE09/AE0A/AE0D/AE0E
  TranslateMonitorCapabilityCodeToBitIndex(0x36) returns internal bit 56
  MonitorCaps_SetCapabilityBit() sets capability_mask[6] |= 0x02
  DisplayCapabilityService passes capability_mask[6] bit 1 as force=1
  DP-sink EDID refresh reruns ordinary I2C-over-AUX EDID reads
```

So the first 5K-capable BootCamp driver appears to make 5K work by forcing a fresh EDID read for selected Apple internal panels, then letting the existing DisplayID/tile/SLS machinery consume the fresh data. It does not appear, in this binary, to add a new Apple-specific DPCD panel wake such as a `0x4f1` write.
