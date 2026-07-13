# ipaopt Architecture Overview

## Overview

Apple's asset catalog pipeline is fundamentally **one way**.

Raw `.xcassets` directories are compiled by Xcode (`actool`) into a binary `Assets.car` archive.

```text
┌──────────────────────┐
│ Raw .xcassets Source │
└──────────┬───────────┘
           │
           │ Xcode / actool
           ▼
┌──────────────────────┐
│   Compiled Assets.car│
└──────────────────────┘
```

Once compiled, an `Assets.car` file becomes an optimized binary blob.

It **cannot** be selectively edited, pruned, or rebuilt without returning to the original asset catalog.

Because of this limitation, **ipaopt** operates at two different stages of the build pipeline depending on what you're trying to optimize.

---

# Mode 1 — `ipaopt strip`

**Target:** Already-built `.ipa` files

This mode performs post-build optimization by removing unused loose resources and thinning Mach-O binaries.

> **Note:** `Assets.car` is left untouched because compiled asset catalogs cannot be modified.

---

## Processing Pipeline

```text
                 Target .ipa
                     │
                     ▼
        Extract Application Bundle
                     │
        ┌────────────┴────────────┐
        ▼                         ▼
 Loose Bundle Resources      Mach-O Binaries
        │                         │
        ▼                         ▼
 Match Removal Rules       Thin CPU Architectures
        │                  (using lipo)
        ▼                         ▼
 Delete Matching Files     Replace Fat Binary
        └────────────┬────────────┘
                     ▼
              Repackage IPA
                     │
                     ▼
          ⚠ Signature Invalidated
           Application must be
           re-signed before use
```

---

## Processing Steps

### 1. Extract Bundle

The IPA archive is unpacked so its application bundle can be inspected.

---

### 2. Strip Loose Resources

ipaopt scans for image files that are **not** stored inside `Assets.car`.

Typical filename conventions include:

- `@1x`
- `@2x`
- `@3x`
- `~ipad`
- `~iphone`

Files matching the configured removal rules (`--remove-idiom`, `--keep-scales`, etc.) are deleted.

---

### 3. Thin Mach-O Binaries

Executables and embedded frameworks are inspected.

Using Apple's `lipo` utility, unwanted CPU slices (for example `armv7` or simulator architectures) are removed while preserving only the requested architectures (such as `arm64`).

---

### 4. Repackage

The modified application bundle is compressed back into a new `.ipa`.

---

## Signature Warning

Any modification to an application bundle invalidates Apple's code signature.

The resulting IPA **must be re-signed** before it can be:

- installed on a device
- sideloaded
- distributed
- submitted to Apple

---

# Mode 2 — `ipaopt catalog-filter`

**Target:** Raw `.xcassets` directories

Unlike `strip`, this mode works **before** compilation.

Since the source assets are still editable, ipaopt can permanently reduce the size of the resulting `Assets.car`.

---

## Processing Pipeline

```text
             Raw .xcassets
                   │
                   ▼
        Scan Asset Catalog Tree
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
   Parse Contents.json    Read Image Files
        │                     │
        ▼                     ▼
 Remove Metadata Entries Delete Unreferenced Assets
        └──────────┬──────────┘
                   ▼
      Write Optimized Asset Catalog
                   │
          --compile specified?
              │           │
              │ No        │ Yes
              ▼           ▼
         Finished     Invoke actool
                           │
                           ▼
                  Optimized Assets.car
```

---

## Processing Steps

### 1. Scan Asset Catalog

ipaopt recursively walks the `.xcassets` directory tree.

Every asset set is inspected, including:

- Image Sets
- App Icons
- Data Sets
- Color Sets
- Symbol Sets

---

### 2. Parse Metadata

Each asset set contains a `Contents.json` file describing available variants.

Examples include:

- device idiom
- image scale
- appearance
- localization

ipaopt parses these metadata files and removes entries that match the configured filters.

Examples:

- remove iPad variants
- remove dark mode assets
- remove unused scales

---

### 3. Remove Unreferenced Files

After metadata has been updated, ipaopt compares the remaining JSON references against the physical files.

Any orphaned assets are deleted automatically.

Supported file types include:

- PNG
- JPEG
- PDF
- SVG (where applicable)

---

### 4. Write Filtered Catalog

The optimized asset catalog is written back to disk.

At this point the catalog remains fully editable and can still be opened in Xcode.

---

### 5. Optional Compilation

When the `--compile` flag is provided, ipaopt invokes Apple's `actool` compiler.

The optimized asset catalog is compiled into a smaller `Assets.car` using Apple's official build tools.

---

# Summary

| Mode | Input | Output | Assets.car Modified |
|------|-------|--------|---------------------|
| `strip` | Built `.ipa` | Optimized `.ipa` | ✖ No |
| `catalog-filter` | `.xcassets` | Filtered source or new `Assets.car` | ✔ Yes |

---

# Key Difference

`strip` optimizes **compiled applications**.

- Removes loose resources
- Thins Mach-O binaries
- Requires re-signing

`catalog-filter` optimizes **source assets**.

- Removes unwanted asset variants
- Deletes unused files
- Produces a smaller `Assets.car`
- Preserves Apple's official asset compilation pipeline
