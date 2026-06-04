# Binary Tools

## UPX — executable compressor

```bash
# Install
scoop install upx

# Compress executable
upx --best myapp.exe

# List compressed info
upx -l myapp.exe

# Decompress
upx -d myapp.exe

# Test compression (don't modify)
upx -t myapp.exe

# Force compression (even for large files)
upx --best --force myapp.exe
```

**Warning:** UPX-packed binaries can trigger antivirus false positives. Always test after packing. Don't pack debug builds.

Typical compression ratios:
- C++ Release binary: 40-60% reduction
- With debug symbols: 60-80% (strip first!)

## Strip + pack pipeline

```bash
# Strip debug symbols (if you don't need them)
strip myapp.exe

# Then compress
upx --best myapp.exe
```

## pe-bear — PE Viewer

```bash
# Installed via scoop: pe-bear
pe-bear myapp.exe
```

Use for:
- Inspect DLL imports/exports
- View PE sections (.text, .data, .rdata)
- Check compiler/linker version
- Verify ASLR, DEP, other security flags
- Check TLS callbacks
- Resource viewer

## depends — Dependency Walker

```bash
# Installed via scoop: depends
depends myapp.exe
```

Shows:
- Which DLLs your binary links against
- Missing DLLs (highlighted in red)
- Import/export functions per DLL
- Delay-loaded dependencies

## ImHex — Hex Editor

```bash
# Installed via scoop: imhex
imhex myapp.exe
```

Features:
- Pattern language for structure definitions
- Disassembler view
- Data inspector (decode values at cursor)
- Diffing
- Bookmarks + annotations
- Dark theme

## Sysinternals

```bash
# Installed via scoop: sysinternals

# Process Monitor — see file/registry/network activity
procmon

# Process Explorer — detailed process info
procexp

# Strings — extract strings from binary
strings myapp.exe
strings -n 8 myapp.exe  # min length 8
strings myapp.exe | rg "http"  # find URLs

# Handle — see open handles
handle -p myapp.exe

# ListDLLs — see loaded DLLs at runtime
listdlls myapp.exe
```

## radare2 / cutter

```bash
# Install
scoop install radare2 cutter

# Quick binary analysis
r2 -A myapp.exe  # auto-analyze

# Inside r2:
# aaaa  — full analysis
# afl   — list functions
# iz    — list strings
# ii    — imports
# ie    — entry points
# VV    — visual mode (graph view)
# pdf @main  — disassemble function

# Cutter (GUI)
cutter myapp.exe
```

## Ghidra

```bash
# Installed via scoop: ghidra
# Launch from Start Menu
```

Full reverse engineering suite:
- Decompiler (machine code → C)
- Function graph
- Cross-references
- Type recovery
- Scripting (Python/Java)
- Diffing

## x64dbg scripting

```
// x64dbg script — find all calls to a function
// Save as script.txt, run from x64dbg
findall call, function_address
```

## cheat-engine

```bash
# Installed via scoop: cheat-engine
# GUI tool for memory scanning/editing at runtime
```

Useful for:
- Finding memory addresses of variables at runtime
- Understanding memory layout of your app
- Reverse engineering game state

## Binary diffing

```bash
# Compare two builds
# Windows
fc /b old.exe new.exe

# Better: use hex editor diff feature
imhex old.exe
# then File → Compare → new.exe
```

## Useful checks before shipping

```bash
# 1. Check dynamic dependencies
depends myapp.exe

# 2. Check security features
pe-bear myapp.exe
# Look for: ASLR (DYNAMIC_BASE), DEP (NX_COMPAT)

# 3. Check for debug symbols leakage
strings myapp.exe | rg "\.pdb"

# 4. Check for hardcoded secrets
strings myapp.exe | rg -i "password|secret|token|key"

# 5. Check file size
ls -lh myapp.exe

# 6. Check import table for suspicious DLLs
pe-bear myapp.exe  # Imports tab
```
