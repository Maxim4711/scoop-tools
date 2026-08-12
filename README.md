# scoop-tools

A personal [Scoop](https://scoop.sh) bucket for the handful of programs where the
public buckets are stale, wrong, or not pinned the way I need them.

```powershell
scoop bucket add scoop-tools https://github.com/Maxim4711/scoop-tools
scoop install scoop-tools/openssl scoop-tools/zlib scoop-tools/pyscripter scoop-tools/beetroot
```

Three of these shadow a manifest of the same name in `main`, `extras` or
`scoop-apps`, so **always install them bucket-qualified** (`scoop-tools/openssl`,
not `openssl`). Scoop records the bucket in `install.json`, so `scoop update`
afterwards stays on this bucket by itself.

| app | version | why it is here |
|---|---|---|
| `openssl` | 3.5.7 | `main` tracks the newest series (4.x). This one is **pinned to the 3.5 LTS series**. |
| `zlib` | 1.3.2 | `extras` ships a **32-bit 1.2.12** binary from 2022. This builds 1.3.2 from source with MSVC. |
| `pyscripter` | 5.3.1 | Same upstream as `scoop-apps`, with a fixed download host and persisted settings. |
| `beetroot` | 1.6.6 | The vendor's manifest runs an NSIS setup that installs into `%ProgramFiles%`. This one is a real portable install. |

## openssl — pinned to 3.5 LTS

OpenSSL's release series are not interchangeable and the newest is rarely the one
you want to build against:

| series | LTS | end of support |
|---|---|---|
| 3.0 | yes | 2026-09-07 |
| 3.5 | **yes** | **2030-04-08** |
| 3.6 | no | 2026-11-01 |
| 4.0 | no | 2027-05-14 |

`main/openssl` follows whatever is newest, which currently means 4.0.x. That is a
non-LTS series with about nine months of support left, and 4.x made
`asn1_string_st` opaque and changed `EVP_Digest`'s arity — which breaks source
that compiled fine on 3.x (`luaossl` is the case that forced this bucket).

So this manifest pins **3.5**, the current LTS. Its `checkver` is a script rather
than a regex over a version list:

```powershell
$json.files.PSObject.Properties |
    Where-Object { $_.Name -like 'Win64OpenSSL-*.exe' -and $_.Value.basever -like '3.5.*' } |
    ForEach-Object { [version]$_.Value.basever } |
    Sort-Object -Descending | Select-Object -First 1
```

Excavator therefore picks up 3.5.7 → 3.5.8 → … as slproweb publishes them — i.e.
**security and patch releases only** — and can never wander onto 3.6 or 4.0.
Moving series is a deliberate edit to `$series` in `bucket/openssl.json`, not
something a scheduled job can do to you.

Import libraries are under `lib\VC\<arch>\MD` (dynamic CRT) and `lib\VC\<arch>\MT`
(static CRT); `lib\` holds hardlinks to the MT set so CMake's `FindOpenSSL` works.
Link `/MD` consumers against `lib\VC\x64\MD` explicitly.

## zlib — built from source

There is no official Windows binary for zlib, which is why `extras/zlib` is stuck
on a 32-bit vc14.2 build of 1.2.12. This manifest downloads the 1.3.2 source and
builds it with CMake + MSVC at install time (hence `depends: main/cmake`, plus
Visual Studio Build Tools with the VCTools workload).

**1.3.2 renamed its Windows outputs** — worth knowing if you are migrating:

| | 1.3.1 | 1.3.2 |
|---|---|---|
| shared runtime | `zlib.dll` | `z.dll` |
| import library | `zlib.lib` | `z.lib` |
| static library | `zlibstatic.lib` | `zs.lib` |

That rename incidentally kills a real trap: `zlib.dll` used to collide with the
Lua binding of the same name and shadow it on `package.cpath`. `z.dll` cannot.

Because LuaRocks (and plenty of autotools-flavoured build systems) insist on
finding `zlib.lib`, the manifest also lays down a **static view**:

```
<dir>\static\include\zlib.h, zconf.h
<dir>\static\lib\zlib.lib      <- a copy of zs.lib, i.e. the STATIC library
```

Point LuaRocks at `ZLIB_DIR=<dir>\static` and it links statically with no runtime
DLL. The copy is deliberately *not* dropped into `lib\` — putting an import
library and a static library both called `zlib.lib` in one directory is how you
end up silently linking the wrong one.

## beetroot — a proper manifest

The vendor's manifest downloads `Beetroot_<v>_x64-setup.exe` and *runs the NSIS
installer* from the install script. Problems with that, all fixed here:

1. **It installed into `%ProgramFiles%`, not the Scoop app directory.** The
   previous local workaround ran the setup and then copied the binary back out of
   `%ProgramFiles%` and deleted it — which needs elevation and races the
   installer. This manifest ignores the `.exe` and extracts the **MSI** payload
   instead (`extract_dir: PFiles\Beetroot`), so Scoop gets a genuine portable
   install with no installer run and no elevation.
2. **The install script hardcoded the filename** (`Beetroot_1.6.5_x64-setup.exe`).
   Autoupdate bumps `version` and the URL but not the script body, so the first
   automatic update would have produced a manifest that installs nothing. There is
   no filename in the script any more because there is no script.
3. **No `bin` entry**, so nothing was on `PATH`. Added.
4. **Uninstall could not remove a running app.** A clipboard manager holds its own
   `.exe` open; `pre_uninstall` now stops the process first, so `scoop update` and
   `scoop uninstall` both work while it is running.
5. **Autostart registry entry.** Beetroot registers itself under `Run`;
   `pre_uninstall` removes it on uninstall so an uninstalled app leaves no
   dangling autostart value.

`%APPDATA%\Beetroot` (database and settings) is outside Scoop's control: it
survives updates and is intentionally left behind on uninstall.

## Automation

* **`.github/workflows/excavator.yml`** — daily; runs Scoop's `checkver.ps1 -Update`
  over `bucket/` and commits version + hash bumps. Also runnable on demand for a
  single app via *Run workflow*.
* **`.github/workflows/ci.yml`** — on push/PR: JSON well-formedness, Scoop
  canonical formatting (`formatjson.ps1`), and liveness of every download URL.
  Full hash verification runs on PRs and on demand, since it downloads everything.

## Licence

Manifests are MIT (see `LICENSE`). The packaged software keeps its own licence —
notably **beetroot is proprietary**; this bucket only describes where to get it.
