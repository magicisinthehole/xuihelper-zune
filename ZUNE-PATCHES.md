# XUIHelper: Zune fork

Vendored from https://github.com/SGCSam/XUIHelper at commit `c0d0830`.

This fork adds Zune HD support to XUIHelper's XUR v5 path. Zune HD shares the
XUR v5 container format with Xbox 360 build 9199 but uses a different XUI class
schema: the per-class property-definition tables differ in membership, order,
and type. The patches below replace the 9199 schema with Zune HD's and fix three
writer bugs that prevented byte-exact recompilation.

## Status

The full 195-file Zune HD v4.5 firmware corpus round-trips:

- **Decompile (`.xur` to `.xui`):** 195/195 with zero warnings or errors. Every
  class is runtime-verified against device memory; there are no parser fallbacks.
- **Recompile (`.xui` to `.xur`):** 189/195 byte-identical, 6/195 byte-different
  but semantically identical (VECT5/QUAT5 lookup-table indices reorder on rebuild;
  the values resolve through the index, so the device renders identical pixels),
  0/195 data corruption.

## Build

`dotnet 8` required. From this directory:

```bash
dotnet build XUIHelper.CLI/XUIHelper.CLI.csproj -c Release
```

CLI is at `XUIHelper.CLI/bin/Release/net8.0/XUIHelper.CLI.dll`.

## Usage

```bash
dotnet XUIHelper.CLI/bin/Release/net8.0/XUIHelper.CLI.dll \
    conv -s input.xur -f xuiv12 -o output.xui -g V5
```

Optional: `-l log.txt -v verbose` for diagnostic logs.

## Changes from upstream

### 1. Cross-platform extension path (`XUIHelper.CLI/Program.cs`)

Upstream hard-codes `Assets\Extensions` with a backslash. Replaced with
`Path.Combine` so the CLI runs on macOS and Linux unmodified.

### 2. Zune HD V5 schema (`Assets/Extensions/V5/`)

The V5 schema is rebuilt from Zune HD's runtime property-definition tables:

- `XuiElements.xml` (43 classes) holds the core hierarchy, replacing upstream's
  9199 layout. Class membership, PropDef order, and value types all match the
  device.
- `ZuneCustomElements.xml` (8 classes) adds Zune HD classes absent from 9199:
  `hdLabel`, `hdNumber`, `XuiNineGrid1`, `XuiKeyStrip`, `XuiCandidateList`,
  `XuiHorzCandidateList`, `XuiMultilineEdit`, `XuiMultilineTextPresenter`.
- The Xbox 360 dashboard schema files (`9199*.xml`) are removed; none of those
  classes exist on Zune HD.
- `9199.xhe` had `IgnoreProperty` entries that, against the Zune schema, matched
  and suppressed 2,292 real properties; they are removed.

### 3. Byte-exact recompile fixes

Three writer bugs corrupted round-tripped output; all are Xbox-360 conventions
that differ on Zune HD:

- **EaseScale default** (`XUI12ReadExtensions.cs`): the reader hard-coded 50
  (Xbox 360), but Zune HD uses 0. Linear keyframes with `EaseScale=0` lost the
  value on round-trip. The reader now parses any ease elements present regardless
  of interpolation type and defaults missing values to 0, matching the writer's
  omission contract.
- **CountHeader presence** (`XUR5.cs`, `XUI.cs`, `XUIHelperAPI.cs`): XUR5 has a
  Flags bit for whether a CountHeader section is present, decided by a heuristic
  calibrated for Xbox 360 dashboards; it mispredicted 8/195 Zune HD files. Two
  changes: the reader's Flags bit is serialized as an `xur5CountHeader` attribute
  on the XUI root and consumed on recompile (so byte-exact round-trips survive),
  and the heuristic is recalibrated for Zune HD (total static plus keyframe
  property values `>= 700`, a clean discriminator: flag=0 max 633, flag=1 min 768)
  for from-scratch authoring where no source flag exists.
- **UTF-16 string write** (`BinaryWriterExtensions.cs`): the string writer (which
  emits UTF-16 BE) used `BinaryWriter.Write(char)`, applying the writer's UTF-8
  encoding. Non-ASCII characters became 2-byte UTF-8 sequences after the 0x00 high
  byte, misaligning the stream. Switched to explicit `Encoding.BigEndianUnicode`.

## How the schema was derived

Zune HD builds each class's property-definition table at runtime in `xuidll.dll`'s
BSS, so the layout cannot be recovered statically. The tables were read from a
live device's memory (class-info arrays for the xuidll classes; gemstone-side
class_info for `XuiKeyStrip`) and transcribed into the V5 schema.