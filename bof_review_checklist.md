## BOF: toastnotify

**Description**: Windows toast notification BOF using WinRT APIs — enumerate AUMIDs, send simple/custom toasts
**Files Reviewed**: entry.c, beacon.h, winrt_defs.h, Makefile, toastnotify.cna, toastnotify_bof.s1.py, toastnotify_custom_bof.s1.py
**Review Status**: ✅ Complete

### Executive Summary

| Severity | Count |
|----------|-------|
| 🔴 Critical | 1 |
| 🟠 High | 2 |
| 🟡 Medium | 2 |
| 🟢 Low | 1 |

**Assessment**: One memory leak in the cleanup path and two stack usage concerns. Core WinRT COM cleanup is well-structured with proper goto-based error handling.

**Recommendation**: 🟠 FIX REQUIRED — address the critical memory leak and high items before operational use.

### Automated Lint Results

```
$ python3 $BOFLINT toastnotify.o --loader any -v
[PASS] Valid entry point: go
[PASS] All relocations supported (IMAGE_REL_AMD64_REL32, IMAGE_REL_AMD64_ADDR32NB)
[PASS] All imports resolvable (KERNEL32, ADVAPI32, MSVCRT, COMBASE, CRYPT32 + Beacon API)
[PASS] No stack-probing symbols (__chkstk_ms)
[WARN] Section '.rdata' is present — not all loaders support read-only/const data
```

The `.rdata` warning is expected — the static `const` GUIDs and runtime class name strings in `winrt_defs.h` land in `.rdata`. Both CS and OC2 loaders handle `.rdata` fine. Merging into `.data` via `objcopy --rename-section` would silence the warning but is unnecessary here.

### Top Findings

#### 🔴 CRITICAL (1)

**1. [Memory Safety] xmlUtf8 not freed on success path** `entry.c:290-292`

In `sendToastCustom`, `xmlUtf8` is heap-allocated at line 251 but only freed on error paths. The success path at line 290 frees `xmlWide` but never frees `xmlUtf8`, leaking `xmlLen + 1` bytes per invocation. Each call to `toast-custom` leaks the decoded XML buffer.

#### 🟠 HIGH (2)

**1. [Code Efficiency] Stack-allocated wchar_t buffers in go()** `entry.c:324, 343`

The `sendtoast` path allocates `wchar_t aumid[256] + title[256] + text_buf[1024]` on the stack — totaling ~3KB. The `custom` path adds another `wchar_t aumid[256]`. While under the 4KB `__chkstk_ms` threshold individually, these are inside `go()` which also has the `datap parser` struct. Combined stack usage is close to the limit. If `go()` grows, it will trigger `__chkstk_ms`. Consider heap-allocating the `text_buf[1024]` (2KB alone).

**2. [Memory Safety] Cleanup label ordering allows use-after-free** `entry.c:198-208`

The goto cleanup labels in `sendToastXml` are ordered: `fail_xmlio` → `fail_xmldoc` → `fail_manager` → `fail_toast` → `fail_early`. But `fail_toast` (line 204-205) releases `toast` which may not have been initialized if the failure occurred before `CreateToastNotification`. The `toast` pointer is initialized to `NULL` at line 163, so the NULL check prevents a crash. However, the label ordering is counterintuitive — `fail_manager` falls through to `fail_toast`, meaning a failure after getting `toastManager` but before creating `toast` will attempt to release a NULL `toast`. The `if (toast)` guard saves it, but the cleanup would be clearer if `fail_toast` came before `fail_manager` in the label chain.

#### 🟡 MEDIUM (2)

**1. [API Usage] Unused DECLSPEC_IMPORT declaration** `entry.c:27`

`ADVAPI32$RegEnumValueW` is declared but never called. Dead declarations increase the review surface and may confuse future maintainers.

**2. [Compiler Compatibility] .rdata section present** `boflint output`

Static const data (GUIDs, runtime class names) generates a `.rdata` section. While functional on CS and OC2, some alternative BOF loaders may not handle `.rdata`. If broader loader compatibility is needed, add `--rename-section .rdata=.data` to the Makefile strip step.

#### 🟢 LOW (1)

**1. [Coding Standards] Mixed declaration styles for API imports** `entry.c:11-18`

Lines 11-16 use `WINBASEAPI` for KERNEL32/MSVCRT declarations while lines 18+ use `DECLSPEC_IMPORT`. Both work, but mixing styles is inconsistent. `DECLSPEC_IMPORT` is the standard BOF convention.

### Review Coverage

| # | Category | Result | Notes |
|---|----------|--------|-------|
| 1 | Project Structure & Naming | ✅ Pass | |
| 2 | Coding Standards | 🟢 Issues | 1 low (L1) |
| 3 | Documentation | ✅ Pass | Clear function comments throughout |
| 4 | API Usage & Declarations | 🟡 Issues | 1 medium (M1) |
| 5 | Beacon API Usage | ✅ Pass | Correct BeaconDataParse/Extract usage |
| 6 | Code Efficiency | 🟠 Issues | 1 high (H1) |
| 7 | Memory Safety & Stability | 🔴 Issues | 1 critical (C1), 1 high (H2) |
| 8 | String & Memory Operations | ✅ Pass | Proper MSVCRT usage, bounds-checked |
| 9 | Global Variables | ✅ Pass | No mutable globals |
| 10 | Argument Parsing | ✅ Pass | Order matches CNA and OC2 scripts |
| 11 | Compiler Compatibility & Build | 🟡 Issues | 1 medium (M2) |
| 12 | Task Appropriateness | ✅ Pass | Quick COM calls, no long-running ops |
| 13 | Conversion Quality | N/A | Original C code |
| 14 | Security Considerations | ✅ Pass | No sensitive data handling |

### OC2 Script Review

**toastnotify_bof.s1.py** — `toast-getaumid` and `toast-sendtoast`
- ✅ Argument encoding order matches `go()` parser: `z` for getaumid, `zzzz` for sendtoast
- ✅ `base_binary_name="toastnotify"` matches compiled BOF filename
- ✅ Quote stripping handles OC2's argument preservation

**toastnotify_custom_bof.s1.py** — `toast-custom` (file upload)
- ✅ Separated into own file for GUI element rendering
- ✅ `validate_files` checks upload before execution
- ✅ Base64 encoding of XML content matches BOF's `sendToastCustom` expectations
- ✅ Argument encoding order: `zzz` (cmd, aumid, b64xml) matches `go()` parser

**toastnotify.cna** — Cobalt Strike
- ✅ `bof_pack` format strings match BOF argument parsing exactly
- ✅ Validation with `berror` on missing args
- ⚠️ `script_resource("build/toastnotify.x64.o")` — path must match deployment layout

### Detailed Category Breakdown

#### 1. Project Structure & Naming

✅ PASS: Main source file is `entry.c`

✅ PASS: BOFNAME `toastnotify` in Makefile matches output filename

✅ PASS: Standard structure — entry.c, beacon.h, winrt_defs.h, Makefile

#### 2. Coding Standards

🟢 LOW (L1): Mixed `WINBASEAPI` and `DECLSPEC_IMPORT` declarations (entry.c:11-18)

✅ PASS: Consistent function naming (camelCase throughout)

✅ PASS: Good const correctness on pointer parameters

#### 3. Documentation

✅ PASS: Functions have clear purpose comments (entry.c:43, 97, 102, 206, 233)

✅ PASS: Argument parsing documented with format string comments

#### 4. API Usage & Declarations

✅ PASS: All Windows APIs declared with `DECLSPEC_IMPORT` or `WINBASEAPI`

🟡 MEDIUM (M1): `ADVAPI32$RegEnumValueW` declared but unused (entry.c:27)

✅ PASS: Proper wide string handling with `MultiByteToWideChar`

✅ PASS: No hardcoded addresses

#### 5. Beacon API Usage

✅ PASS: Correct `BeaconDataParse` / `BeaconDataExtract` usage (entry.c:300-301)

✅ PASS: Proper callback types — `CALLBACK_OUTPUT` for results, `CALLBACK_ERROR` for failures

✅ PASS: No standard library functions — uses MSVCRT equivalents

#### 6. Code Efficiency

🟠 HIGH (H1): Stack buffers in `go()` total ~3KB, close to the 4KB threshold (entry.c:324, 343)

✅ PASS: Single-threaded execution

✅ PASS: Short execution — COM calls complete quickly

✅ PASS: `enumSubkeys` loop is bounded by registry API return values (ERROR_NO_MORE_ITEMS)

✅ PASS: Symbols stripped with `--strip-unneeded`

#### 7. Memory Safety & Stability

🔴 CRITICAL (C1): `xmlUtf8` leaked on success path in `sendToastCustom` (entry.c:290-292)

🟠 HIGH (H2): Cleanup label ordering in `sendToastXml` is counterintuitive (entry.c:198-208)

✅ PASS: All HeapAlloc return values checked (entry.c:216, 252, 276)

✅ PASS: Registry keys closed on all paths (entry.c:71)

✅ PASS: COM objects properly Released with NULL guards

✅ PASS: `RoUninitialize` called on all exit paths when `roInitCalled` is TRUE

✅ PASS: `BeaconFormatFree` called after use (entry.c:79)

#### 8. String & Memory Operations

✅ PASS: Uses `MSVCRT$_snwprintf` with size limit for XML building (entry.c:221)

✅ PASS: Null-terminates buffer after `_snwprintf` (entry.c:231)

✅ PASS: `CryptStringToBinaryA` uses sizing call before allocation (entry.c:239-244)

✅ PASS: `MultiByteToWideChar` uses sizing call before allocation (entry.c:267)

#### 9. Global Variables

✅ PASS: No mutable globals — only `static const` data in `winrt_defs.h`

#### 10. Argument Parsing

✅ PASS: `go()` parses cmd first, then subcommand-specific args — matches CNA and OC2 packing order

✅ PASS: NULL checks on extracted arguments before use (entry.c:303, 319, 338)

✅ PASS: Cross-validated: CNA `bof_pack("zzzz", ...)` matches OC2 `[STR, STR, STR, STR]` matches `go()` extract order

#### 11. Compiler Compatibility & Build

🟡 MEDIUM (M2): `.rdata` section present — functional but may cause issues with alternative loaders

✅ PASS: x64 target compiled and stripped

✅ PASS: No external library dependencies beyond standard DLLs

#### 12. Task Appropriateness

✅ PASS: WinRT toast notification is appropriate for BOF — quick COM activation, no long-running ops

#### 13. Conversion Quality

N/A — Original C code

#### 14. Security Considerations

✅ PASS: Input validated — NULL checks on all BeaconDataExtract results

✅ PASS: No sensitive data (passwords, keys) handled

✅ PASS: No thread-safety concerns — single-threaded execution
