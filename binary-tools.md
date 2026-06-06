# Binary Tools

## Quick Reference

| Tool | Install | What |
|------|---------|------|
| **UPX** | `scoop install upx` | Compress executables |
| **pe-bear** | `scoop install pe-bear` | PE viewer |
| **depends** | 🔧 [dependencywalker.com](https://www.dependencywalker.com) | DLL dependencies |
| **ImHex** | `scoop install imhex` | Hex editor |
| **checksec** | `scoop install checksec` | Security audit |
| **strings** | via binutils / sysinternals | Extract strings |

> Full RE tools: [reverse-engineering.md](reverse-engineering.md)

## UPX — Compress Executables

```bash
upx --best myapp.exe         # compress
upx -l myapp.exe             # info
upx -d myapp.exe             # decompress
strip myapp.exe && upx --best myapp.exe   # strip + pack
```

⚠️ Can trigger AV false positives. Always test after packing. Don't pack debug builds.

## pe-bear — PE Viewer

```bash
pe-bear myapp.exe
```

Inspect: sections, imports/exports, security flags (ASLR, DEP), compiler version, TLS callbacks.

## depends — DLL Dependencies

```bash
depends myapp.exe              # Windows GUI
ldd myapp                      # Linux (built-in)
otool -L myapp                 # macOS (built-in)
```

Shows: required DLLs, missing DLLs (red), import/export functions.

## ImHex — Hex Editor

```bash
imhex myapp.exe
```

Pattern language for structure definitions, disassembler, data inspector, diffing.

## Sysinternals

```bash
scoop install sysinternals     # install all

strings myapp.exe                # extract strings
strings -n 8 myapp.exe           # min length 8
strings myapp.exe | rg "http"    # find URLs
procmon                          # file/registry/network monitor
procexp                          # process explorer
handle -p myapp.exe              # open handles
```

## Pre-Ship Checklist

```bash
# 1. Security features
checksec --file=myapp              # ASLR, DEP, PIE, stack canary

# 2. DLL dependencies
depends myapp.exe                  # Windows
ldd myapp                          # Linux
otool -L myapp                     # macOS

# 3. Security flags
pe-bear myapp.exe                  # check ASLR (DYNAMIC_BASE), DEP (NX_COMPAT)

# 4. Debug symbol leakage
strings myapp.exe | rg "\.pdb"
nm myapp | rg "debug_info"        # ELF

# 5. Hardcoded secrets
strings myapp.exe | rg -i "password|secret|token|key"

# 6. File size
ls -lh myapp.exe

# 7. Compress
strip myapp.exe && upx --best myapp.exe

# 8. Re-check after packing
depends myapp.exe
```

> 💡 Run the pre-ship checklist before every release. Five minutes saves hours of debugging customer issues.

---

→ **Reverse Engineering**: [reverse-engineering.md](reverse-engineering.md) — Ghidra, r2, x64dbg
→ **Install**: [tools-install.md](tools-install.md) — install all binary tools
→ **File tracking**: README.md section 10 — trace what files a command produces
