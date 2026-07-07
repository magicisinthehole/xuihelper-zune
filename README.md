# XUIHelper (Zune HD fork)

A C# suite for reading, writing, and converting XUI/XUR UI files. This is a fork
of [SGCSam/XUIHelper](https://github.com/SGCSam/XUIHelper) (vendored at commit
`c0d0830`) that adds full Zune HD support to the XUR v5 path.

Zune HD shares the XUR v5 container format with Xbox 360 build 9199 but uses a
different XUI class schema: the per-class property-definition tables differ in
membership, order, and type. This fork replaces the 9199 schema with Zune HD's,
extracted from device runtime memory, and fixes the writer bugs that prevented
byte-exact recompilation.

## Zune HD status

The full 195-file Zune HD v4.5 firmware corpus round-trips:

- **Decompile (`.xur` to `.xui`):** 195/195, zero warnings or errors. Every class
  is runtime-verified against device memory; there are no parser fallbacks.
- **Recompile (`.xui` to `.xur`):** 189/195 byte-identical, 6/195 byte-different
  but semantically identical (VECT5/QUAT5 lookup-table indices reorder on rebuild;
  the values resolve through the index, so the device renders identical pixels),
  0/195 data corruption.

See [`ZUNE-PATCHES.md`](ZUNE-PATCHES.md) for the full list of changes from upstream
and how the schema was derived.

## Build

`dotnet 8` required. From this directory:

```bash
dotnet build XUIHelper.CLI/XUIHelper.CLI.csproj -c Release
```

The CLI is at `XUIHelper.CLI/bin/Release/net8.0/XUIHelper.CLI.dll`.

## Usage

Convert a Zune HD `.xur` to editable `.xui`:

```bash
dotnet XUIHelper.CLI/bin/Release/net8.0/XUIHelper.CLI.dll \
    conv -s input.xur -f xuiv12 -o output.xui -g V5
```

Recompile a `.xui` back to `.xur`:

```bash
dotnet XUIHelper.CLI/bin/Release/net8.0/XUIHelper.CLI.dll \
    conv -s output.xui -f xurv5 -o rebuilt.xur -g V5
```

`massconv` converts a whole directory; `-l log.txt -v verbose` writes diagnostic
logs. Full CLI reference: [`Documentation/XUIHelper.CLI Documentation.md`](Documentation/XUIHelper.CLI%20Documentation.md).

## Capabilities

Inherited from upstream:

- Read/write support for XUR v5, XUR v8 (builds 12611 - 17559), and XUI v12
- Interoperability between XUR v5, XUR v8, and XUI v12
- An XML extensions system for registering custom classes ([`Documentation/XML Extensions Documentation.md`](Documentation/XML%20Extensions%20Documentation.md))
- A core library (`XUIHelper.Core`) with both CLI and GUI wrappers

## Scope of the fork

This fork retargets the XUR v5 path to Zune HD. The bundled V5 extensions
(`Assets/Extensions/V5/`) are Zune HD's schema, and the XUR5 count-header write
heuristic is calibrated for the Zune corpus, so **this fork does not target Xbox
360 build 9199 v5 out of the box** (upstream's 9199 V5 schema is removed). Use
upstream XUIHelper for Xbox 360 v5 work.

The XUR v8 and XUI v12 machinery is retained from upstream and is not the focus of
this fork. Two of the byte-exact fixes touch code shared with those paths: the
UTF-16 string writer is used only by the V5 string section, but the XUI v12 reader
change (keyframe ease defaults, 50 to 0) applies whenever a `.xui` is read,
including `.xui` to XUR v8 recompiles.

## Credits

XUIHelper is by [SGCSam](https://github.com/SGCSam). The original XuiWorkshop work
by MaesterRowen and Wondro, and the additional XML extensions from jaydergham, made
it possible. The Zune HD schema and round-trip fixes in this fork were reversed from
device runtime memory (see `ZUNE-PATCHES.md`).
