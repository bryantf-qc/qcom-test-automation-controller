# Build and Run Guide — TACDev.dll and qtac-app.exe

## Prerequisites

| Tool | Version | Notes |
|------|---------|-------|
| Visual Studio Build Tools | 2022 | MSVC x64 toolchain (`cl.exe`) |
| CMake | 3.22+ | Must be on `PATH` |
| Ninja | any | Must be on `PATH` |
| Qt | 6.11.1 | Installed to `C:\Qt\6.11.1\msvc2022_64\` |
| FTDI CDM driver SDK | 2.12.36+ | See §FTDI Bootstrap below |

---

## FTDI Bootstrap (one-time)

The build requires `ftd2xx.lib` and `ftd2xx.dll` placed under `__Builds\x64\`.
Copy them from an existing checkout or from your FTDI CDM driver installation:

```powershell
# From an existing qcom-test-automation-controller checkout at the default location:
Copy-Item C:\ProdTools\qcom-test-automation-controller\__Builds\x64\Release\lib\ftd2xx.lib `
          __Builds\x64\Release\lib\ -Force
Copy-Item C:\ProdTools\qcom-test-automation-controller\__Builds\x64\Release\bin\ftd2xx.dll `
          __Builds\x64\Release\bin\ -Force
Copy-Item C:\ProdTools\qcom-test-automation-controller\__Builds\x64\Debug\lib\ftd2xx.lib `
          __Builds\x64\Debug\lib\  -Force
Copy-Item C:\ProdTools\qcom-test-automation-controller\__Builds\x64\Debug\bin\ftd2xx.dll `
          __Builds\x64\Debug\bin\  -Force
```

Or copy directly from the FTDI CDM driver package (`amd64\ftd2xx.lib`, `ftd2xx.dll`).

---

## Build

### Quick build (recommended) — PowerShell

```powershell
cd C:\ProdTools\qcom-test-automation-controller
.\build_app.ps1
```

Performs:
1. Locates VS2022 via `vswhere.exe` and loads the MSVC x64 environment
2. Adds Qt `bin\` to `PATH`
3. `cmake -S . -B build\Release -DCMAKE_PREFIX_PATH=<Qt> -G Ninja -DCMAKE_BUILD_TYPE=Release`
4. `cmake --build build\Release --target qtac-app`
5. `cmake --build build\Release --target TACDev`

Outputs:
```
build\Release\qtac-app.exe
build\Release\TACDev.dll
build\Release\TACDev.lib   (import lib for callers)
```

### Quick build — batch file

```bat
cd C:\ProdTools\qcom-test-automation-controller
build_app.bat
```

Same steps as the PowerShell script; build log is saved to `build_out.txt`.

### Manual CMake steps

```bat
call "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvars64.bat"
set PATH=C:\Qt\6.11.1\msvc2022_64\bin;%PATH%

cd C:\ProdTools\qcom-test-automation-controller

cmake -S . -B build\Release ^
      -DCMAKE_PREFIX_PATH=C:\Qt\6.11.1\msvc2022_64 ^
      -G Ninja ^
      -DCMAKE_BUILD_TYPE=Release

cmake --build build\Release --target qtac-app TACDev
```

### Building individual targets

```bat
cmake --build build\Release --target qtac-app      # GUI application only
cmake --build build\Release --target TACDev        # DLL only
cmake --build build\Release --target qtac-core     # Qt-free static lib only
```

---

## Rebuild (incremental, no reconfigure)

When only source files change and the build is already configured:

```powershell
cd C:\ProdTools\qcom-test-automation-controller
.\rebuild_library.ps1    # rebuilds qtac-core + TACDev + qtac-app (Debug)
.\rebuild_tests.ps1      # rebuilds test_coders_commands, test_hardware_ftdi, test_hardware_psoc (Debug)
```

---

## Deploy (qtac-app.exe)

After building, run the deploy script to copy Qt runtime DLLs and device
configuration files alongside the executable:

```powershell
cd C:\ProdTools\qcom-test-automation-controller
.\deploy_app.ps1
```

This copies into `build\Release\`:
- `ftd2xx.dll` — FTDI runtime
- `devicelist.json` — device catalogue
- `*.tcnf` — device configuration files
- `DefaultScript.txt` — default AlpacaScript
- All Qt DLLs/plugins (via `windeployqt`)

### Running qtac-app

```bat
build\Release\qtac-app.exe
```

The application expects `devicelist.json` and `.tcnf` files to be **in the
same directory** as the executable. `deploy_app.ps1` places them there.

---

## Deploy (TACDev.dll)

`TACDev.dll` is a Qt-free C API DLL. Consumers need:

| File | Location after build |
|------|---------------------|
| `TACDev.dll` | `build\Release\TACDev.dll` |
| `TACDev.lib` | `build\Release\source\tacdev\TACDev.lib` |
| `TACDev.h` | `source\tacdev\TACDev.h` |
| `ftd2xx.dll` | copy from `__Builds\x64\Release\bin\` |
| `devicelist.json` | `configurations\devicelist.json` |
| `*.tcnf` | `configurations\*.tcnf` |

**At runtime**, `devicelist.json` and all `.tcnf` files must be findable
relative to the executable that loads `TACDev.dll`. The DLL searches starting
from the exe directory and walking up to four levels.

### Minimal C usage example

```c
#include "TACDev.h"
#include <stdio.h>

int main(void) {
    if (InitializeTACDev() != NO_TAC_ERROR) return 1;

    int count = 0;
    GetDeviceCount(&count);
    printf("Devices found: %d\n", count);

    for (int i = 0; i < count; i++) {
        char port[256];
        GetPortData(i, port, sizeof(port));
        printf("  [%d] %s\n", i, port);
    }

    TAC_HANDLE h = OpenHandleByDescription("COM3");  // adjust as needed
    if (h == kBadHandle) { printf("Open failed\n"); return 1; }

    char name[128];
    GetName(h, name, sizeof(name));
    printf("Device: %s\n", name);

    SendCommand(h, "PowerKey", true);

    CloseTACHandle(h);
    return 0;
}
```

---

## Running Tests

### Non-hardware tests (no device required)

```bat
cd C:\ProdTools\qcom-test-automation-controller\build\Release
test_bytearray.exe
test_coders_commands.exe
test_containers.exe
test_platform_configs.exe
test_signal.exe
test_string.exe
test_stringutils.exe
```

### Hardware integration tests (device must be connected)

```bat
cd C:\ProdTools\qcom-test-automation-controller\build\Release
test_hardware_ftdi.exe    # requires a TACLite (FTDI) device
test_hardware_psoc.exe    # requires a PSoC TAC device
test_tacdev_api.exe       # requires any supported TAC device
```

### Rebuild tests only

```powershell
cd C:\ProdTools\qcom-test-automation-controller
.\rebuild_tests.ps1
```

---

## Project Structure (relevant to this guide)

```
C:\ProdTools\qcom-test-automation-controller\
├── CMakeLists.txt              root — adds all subdirectories
├── build_app.ps1               PowerShell build script (uses vswhere)
├── build_app.bat               Batch build script
├── rebuild_library.ps1         Incremental rebuild of library + app targets
├── rebuild_tests.ps1           Incremental rebuild of test targets
├── deploy_app.ps1              windeployqt + copy configs
├── __Builds\x64\               ftd2xx.lib / ftd2xx.dll (bootstrapped)
├── configurations\             devicelist.json, *.tcnf
├── third-party\                boost, hidapi, libserialport, nlohmann/json
├── source\
│   ├── library\                qtac-core Qt-free static lib
│   ├── libraries\qt-adapter\   Qt ↔ qtac-core bridge (static lib)
│   ├── app\                    qtac-app Qt6 GUI source
│   ├── tacdev\                 TACDev.dll C API source + TACDev.h
│   └── test\                   unit + integration tests
└── build\Release\              cmake build output
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `cmake` can't find Qt | Qt not on `CMAKE_PREFIX_PATH` | Pass `-DCMAKE_PREFIX_PATH=C:\Qt\6.11.1\msvc2022_64` |
| `ftd2xx.lib not found` | FTDI not bootstrapped | Copy from CDM driver package or sibling checkout |
| `ftd2xx.dll not found` at runtime | DLL not deployed | Run `deploy_app.ps1` or copy manually |
| App opens but shows no devices | `devicelist.json` missing | Run `deploy_app.ps1` or copy `configurations\` files |
| `cl.exe` not found | MSVC env not loaded | Use `build_app.ps1`/`build_app.bat` which load `vcvars64.bat` |
| Build fails with `LINK : fatal error LNK1181` | Stale build dir | Delete `build\Release` and reconfigure |
| hidapi not found | FetchContent network issue | Ensure internet access during first configure |
