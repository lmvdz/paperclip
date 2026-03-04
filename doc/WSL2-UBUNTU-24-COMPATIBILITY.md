# WSL 2 / Ubuntu 24.04 Embedded PostgreSQL Compatibility

## Problem Summary

When running `pnpm dev` on WSL 2 with Ubuntu 24.04 (Noble), the embedded PostgreSQL binaries fail to initialize with the error:

```
Postgres init script exited with code 1
```

This cryptic error hides the actual underlying issue, which is a missing shared library dependency.

## Root Cause

The `embedded-postgres` npm package uses pre-compiled PostgreSQL binaries from [zonkyio/embedded-postgres-binaries](https://github.com/zonkyio/embedded-postgres-binaries). These binaries were built on older Linux distributions and are **hard-linked against ICU library version 60**.

Ubuntu 24.04 (Noble) ships with **ICU version 74**, and does not include the older ICU 60 libraries for backward compatibility.

When the PostgreSQL `initdb` binary tries to run, it fails with:

```
error while loading shared libraries: libicuuc.so.60: cannot open shared object file: No such file or directory
```

However, this error is hidden by the `embedded-postgres` package, which only reports "Postgres init script exited with code 1".

## Why Symlinking Doesn't Work

A common attempted solution is to create symbolic links:

```bash
sudo ln -s /lib/x86_64-linux-gnu/libicuuc.so.74 /lib/x86_64-linux-gnu/libicuuc.so.60
sudo ln -s /lib/x86_64-linux-gnu/libicui18n.so.74 /lib/x86_64-linux-gnu/libicui18n.so.60
sudo ln -s /lib/x86_64-linux-gnu/libicudata.so.74 /lib/x86_64-linux-gnu/libicudata.so.60
```

However, this fails with:

```
symbol lookup error: /path/to/initdb: undefined symbol: uloc_toLanguageTag_60
```

**Why?** ICU uses **symbol versioning**. The PostgreSQL binaries specifically request symbols with the `@ICU_60` version tag (e.g., `uloc_toLanguageTag_60`). Even though ICU 74 has the same function (`uloc_toLanguageTag`), it's tagged as `@ICU_74`, so the dynamic linker cannot find the requested symbol.

This is a fundamental incompatibility at the binary level.

## Technical Details: ICU Symbol Versioning

ICU (International Components for Unicode) uses symbol versioning to maintain ABI compatibility across versions. When a binary is compiled against ICU 60, it embeds version-specific symbol requests:

```
$ objdump -T /path/to/initdb | grep ICU
0000000000000000      DF *UND*  0000000000000000  ICU_60  uloc_toLanguageTag
0000000000000000      DF *UND*  0000000000000000  ICU_60  ucol_open
```

The `ICU_60` suffix means the binary will **only** accept symbols from ICU version 60. Symlinking a newer library doesn't change these version tags.

## Affected Systems

This issue affects **any Linux system where the system ICU version doesn't match the version the PostgreSQL binaries were compiled against** (ICU 60).

**Confirmed affected:**
- WSL 2 running Ubuntu 24.04 (Noble) - ICU 74
- Native Ubuntu 24.04 installations - ICU 74
- Ubuntu 22.04 (Jammy) - ICU 70
- Ubuntu 20.04 (Focal) - ICU 66
- Debian 13 (Trixie) - ICU 74+
- Fedora 40+ - ICU 74+

**Not affected:**
- **macOS** - Uses system-provided ICU with different versioning scheme; binaries are built separately
- **Windows** - Bundles its own ICU libraries with the PostgreSQL binaries
- **Older Ubuntu versions** (18.04 and earlier) - May have ICU 60 or compatible versions

**Note:** The issue is most commonly reported on Ubuntu 24.04 because it's the newest LTS release that users are actively upgrading to.

## Solutions

### Option 1: Docker PostgreSQL (Recommended)

The fastest and most reliable solution is to run PostgreSQL in a Docker container:

```bash
# Start PostgreSQL container
docker run -d --name paperclip-postgres \
  -e POSTGRES_USER=paperclip \
  -e POSTGRES_PASSWORD=paperclip \
  -e POSTGRES_DB=paperclip \
  -p 5432:5432 \
  postgres:16-alpine

# Set DATABASE_URL and run
export DATABASE_URL='postgres://paperclip:paperclip@localhost:5432/paperclip'
pnpm dev
```

**Advantages:**
- ✅ Works immediately (30 seconds to set up)
- ✅ Isolated from system libraries
- ✅ Matches production environment
- ✅ Easy to reset/recreate

### Option 2: Native PostgreSQL Installation

Install PostgreSQL directly on WSL 2:

```bash
# Install PostgreSQL
sudo apt update
sudo apt install postgresql postgresql-contrib

# Start the service
sudo service postgresql start

# Create user and database
sudo -u postgres createuser -s paperclip
sudo -u postgres createdb -O paperclip paperclip

# Set DATABASE_URL and run
export DATABASE_URL='postgres://paperclip@localhost/paperclip'
pnpm dev
```

**Advantages:**
- ✅ Native performance
- ✅ Uses system ICU (no compatibility issues)
- ✅ Standard PostgreSQL tools work normally

### Option 3: PGlite (Experimental)

PGlite is a WASM-based PostgreSQL that runs in-process:

```bash
# Set environment variable
export DATABASE_MODE=pglite

# Run normally
pnpm dev
```

**Note:** PGlite support may be limited in the current version. Check `packages/db/README.md` for details.

## Why Updated Binaries Don't Exist

The `embedded-postgres-binaries` project would need to:
1. Rebuild PostgreSQL against ICU 74
2. Publish new binary packages for all platforms
3. Update version tags in the npm packages

As of March 2026, **no ICU 74-compatible binaries exist** in the zonkyio repository.

Building binaries on-the-fly is not practical because:
- Compilation takes 15-30 minutes on a fast machine
- Requires hundreds of MB of build dependencies
- Defeats the purpose of "embedded" PostgreSQL
- Would need complex cross-platform build logic

## Recommendation

For production-like development environments, **always use external PostgreSQL**:
- More realistic testing environment
- Better performance and debugging
- No dependency on specific system library versions
- Easier to match production configuration

Embedded PostgreSQL is best suited for:
- Quick prototyping
- CI/CD environments with controlled OS versions
- Systems with compatible ICU versions (older distributions)

## References

- [embedded-postgres npm package](https://www.npmjs.com/package/embedded-postgres)
- [zonkyio/embedded-postgres-binaries](https://github.com/zonkyio/embedded-postgres-binaries)
- [ICU Symbol Versioning](https://unicode-org.github.io/icu/userguide/icu4c/packaging.html)
- [Ubuntu 24.04 Release Notes](https://discourse.ubuntu.com/t/noble-numbat-release-notes/39890)

