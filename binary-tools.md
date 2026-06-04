# Binary Tools

## UPX — compress executables

```bash
upx --best myapp.exe         # compress
upx -l myapp.exe             # info
upx -d myapp.exe             # decompress
strip myapp.exe && upx --best myapp.exe   # strip + pack
```

⚠️ Can trigger AV false positives. Always test after packing. Don't pack debug builds.

## pe-bear — PE viewer

```bash
pe-bear myapp.exe
```

Inspect: sections, imports/exports, security flags (ASLR, DEP), compiler version, TLS callbacks.

## depends — DLL dependencies

```bash
depends myapp.exe
```

Shows: required DLLs, missing DLLs (red), import/export functions.

## ImHex — hex editor

```bash
imhex myapp.exe
```

Pattern language for structure definitions, disassembler, data inspector, diffing.

## Sysinternals

```bash
strings myapp.exe                # extract strings
strings -n 8 myapp.exe           # min length 8
strings myapp.exe | rg "http"    # find URLs
procmon                           # file/registry/network monitor
procexp                           # process explorer
handle -p myapp.exe               # open handles
```

## radare2 / cutter

```bash
r2 -A myapp.exe    # auto-analyze
# Inside: aaaa (full analysis), afl (functions), iz (strings), ii (imports), VV (graph)

cutter myapp.exe   # GUI
```

## Ghidra

```bash
# Launch from Start Menu
```

Full decompiler (machine code → C), function graphs, cross-references, scripting.

## cheat-engine

```bash
# GUI — memory scanner/editor at runtime
```

Useful for: finding variable addresses, understanding memory layout.

## Pre-ship checklist

```bash
# 1. DLL dependencies
depends myapp.exe

# 2. Security flags
pe-bear myapp.exe   # check ASLR (DYNAMIC_BASE), DEP (NX_COMPAT)

# 3. Debug symbol leakage
strings myapp.exe | rg "\.pdb"

# 4. Hardcoded secrets
strings myapp.exe | rg -i "password|secret|token|key"

# 5. File size
ls -lh myapp.exe

# 6. Compress
strip myapp.exe && upx --best myapp.exe

# 7. Re-check after packing
depends myapp.exe
```

> 💡 **Tip:** Run the pre-ship checklist before every release. Five minutes saves hours of debugging customer issues.
