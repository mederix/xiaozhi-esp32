# Building from the command line on Windows

Everything below was executed on Windows 11 against tag `v2.4.2`, board
`spotpear/sp-esp32-s3-1.54-muma`, ESP-IDF **v6.0.2**. No Docker, no WSL.

The CI builds this project inside `espressif/idf:v6.0.2` (a Linux container), so the
Windows path is not exercised there. Three things bite, and each one fails with a message
that points somewhere else. They are listed under [Gotchas](#gotchas).

## What you need

| item | requirement |
|---|---|
| ESP-IDF | the version pinned by `.github/workflows/build.yml` for the tree you are building — `v6.0.2` at `v2.4.2` |
| Python | **any 3.10 or newer**. `tools/python_version_checker.py` in IDF 6 declares `OLDEST_PYTHON_SUPPORTED = (3, 10)` and compares with `>=` only, so there is no upper bound. 3.14 works |
| disk | ~1.2 GB for the IDF source, ~6.3 GB for the toolchain, ~600 MB per build tree |

Install ESP-IDF manually rather than through the installer if you want to control where the
toolchain lands — `IDF_TOOLS_PATH` is the only thing that keeps it off the system drive:

```powershell
git clone -b v6.0.2 --recursive --shallow-submodules https://github.com/espressif/esp-idf.git D:\esp\v6.0.2\esp-idf
$env:IDF_TOOLS_PATH = 'D:\esp\tools'
$IDF = 'D:\esp\v6.0.2\esp-idf'
python "$IDF\tools\idf_tools.py" --idf-path $IDF install --targets=esp32s3
python "$IDF\tools\idf_tools.py" --idf-path $IDF install-python-env --features=core
```

## Activating the environment

`IDF_TOOLS_PATH` is not persistent — set it in **every** new shell, before `export.ps1`, or
the tools are looked up under `%USERPROFILE%\.espressif` instead:

```powershell
$env:IDF_TOOLS_PATH = 'D:\esp\tools'; . 'D:\esp\v6.0.2\esp-idf\export.ps1'
```

Verify with `idf.py --version`.

## Option A — `scripts/build.py`

This is what CI uses, and it applies the board's `sdkconfig_append` for you.

**The positional argument is the `BOARD_DIR` path, not the `type` field of `config.json`.**
For the MUMA board, `config.json` says `"type": "sp-esp32-s3-1.54-muma"`, but
`main/CMakeLists.txt` maps the Kconfig symbol to `BOARD_DIR "spotpear/sp-esp32-s3-1.54-muma"`,
and that is what `_find_board_config_candidates()` matches. Passing the short name fails with:

```
[ERROR] board_type sp-esp32-s3-1.54-muma not found in main/CMakeLists.txt
```

Use `python scripts/build.py --list-boards` — it prints the accepted form:

```powershell
python scripts/build.py spotpear/sp-esp32-s3-1.54-muma --name sp-esp32-s3-1.54-muma
```

## Option B — `idf.py` directly

Useful when you want to see exactly which Kconfig symbol selects the board, or when you are
bisecting a build problem. Write a one-line defaults file:

```
# sdkconfig.muma
CONFIG_BOARD_TYPE_SPOTPEAR_ESP32_S3_1_54_MUMA=y
```

Then:

```powershell
idf.py -DSDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.defaults.esp32s3;sdkconfig.muma" set-target esp32s3 build
```

When you pass `SDKCONFIG_DEFAULTS` explicitly you must list **all** the files, including the
per-target one — ESP-IDF stops appending `sdkconfig.defaults.<target>` on its own.

Find the symbol for any board in `main/Kconfig.projbuild`, or by looking up the `BOARD_DIR`
in `main/CMakeLists.txt`.

## Gotchas

### 1. `[WinError 193]` from `scripts/build.py`

```
File "scripts/build.py", line 62, in _run_idf
    if subprocess.run(command, check=False).returncode != 0:
OSError: [WinError 193] %1 is not a valid Win32 application
```

`_run_idf()` builds `["idf.py", ...]`. On Windows, ESP-IDF exposes `idf.py` through the
`idf-exe` shim (`idf.py.exe`), but `CreateProcess` **does not apply `PATHEXT`** — so the bare
name resolves to the raw script under `$IDF_PATH/tools` and cannot be executed. Measured:

```
shutil.which("idf.py")                      -> ...\idf-exe\1.0.3\idf.py.EXE
subprocess.run(["idf.py", "--version"])     -> OSError [WinError 193]
subprocess.run(["idf.py.exe", "--version"]) -> OK
```

`shutil.which()` *does* apply `PATHEXT`, so resolving through it fixes Windows and changes
nothing on POSIX, where it returns `idf.py` itself:

```python
command = [shutil.which("idf.py") or "idf.py"]
```

This fork carries that patch. Note that it only bites the script — `idf.py` typed into
PowerShell works, because `export.ps1` defines a shell function of that name.

### 2. A leftover `build/` directory blocks `set-target`

If a previous run died before CMake configured anything, `build/` exists but is not a CMake
build directory, and `set-target` refuses to clean it:

```
Directory '...\build' doesn't seem to be a CMake build directory.
Refusing to automatically delete files in this directory.
```

Delete `build/` and run again.

### 3. ⚠️ `idf.py flash` overwrites the assets partition

The command the build prints on success ends with:

```
0x800000 build\generated_assets.bin
```

On a 16 MB board using `partitions/v2/16m.csv`, `0x800000` is the 8 MB `assets` partition.
If that partition holds anything you did not generate from this tree — a custom wake word
written with `assets.mjs`, a model flashed from a vendor console — **the default flash
command replaces it**, without asking and without `erase_flash` being involved.

Read the flash back before your first `idf.py flash` on a device you care about:

```powershell
python -m esptool -p COM7 -b 921600 read-flash 0 0x1000000 backup.bin
```

To flash only the application and leave assets alone, write just the app offset:

```powershell
python -m esptool --chip esp32s3 -b 460800 write-flash 0x20000 build\xiaozhi.bin
```
