# Windows MinGW Build and Debug Setup

## 1. Build Setup for Windows Users using MinGW

These instructions build a native Windows executable. Building for the ARM64
SRS box requires the separate cross-compilation workflow described in README.md.
The existing prerequisites, including submodules, Go, and Bushido, still apply.

### Select the native Windows compiler

CMake originally selected `C:\msys64\usr\bin\cc.exe` and `c++.exe`.
These target the MSYS/Cygwin environment rather than native Windows. With native
Windows CMake and Ninja, the escaped configuration argument caused the compiler
to report `fatal error: no input files` during CMake's initial compiler check.

Use the installed MinGW-w64 compilers under `C:\msys64\mingw64\bin` instead.
Add that directory to the current PowerShell session's PATH so compiler tools
and runtime DLLs can be found:

```powershell
$env:PATH = "C:\msys64\mingw64\bin;$env:PATH"
```

### Configure and build in the same directory

```powershell
cmake -S . -B build-mingw -G "Ninja Multi-Config" -DCMAKE_C_COMPILER=C:/msys64/mingw64/bin/gcc.exe -DCMAKE_CXX_COMPILER=C:/msys64/mingw64/bin/g++.exe -DCMAKE_CONFIGURATION_TYPES="Debug;Release"
cmake --build build-mingw --config Release
```

The corrections and their reasons are:

- Explicit compiler paths prevent CMake from selecting the MSYS compilers again.
- A separate `build-mingw` directory avoids the incorrect compiler paths cached
	by the failed configuration in `build`.
- Remove the stray standalone `build` argument after `"Ninja Multi-Config"` in
	the README command; `-B` already specifies the build directory.
- Use `CMAKE_CONFIGURATION_TYPES`, plural. The singular
	`CMAKE_CONFIGURATION_TYPE` does not select the multi-config configurations.
- Pass `build-mingw` to both commands. Building `build` after configuring
	`build-mingw` caused Ninja's missing `build-Release.ninja` error.

### Keep the Windows PATH scoped appropriately

Adding `C:\msys64\mingw64\bin` to the Windows user PATH is optional.
Restart all VS Code windows afterward to inherit the change. Avoid adding
`C:\msys64\usr\bin` for this workflow. Multiple compiler installations on PATH
can cause tool or DLL conflicts; project-specific settings avoid a global change.

## 2. Debug Setup for Windows Users using MinGW

### Install the debugger and extensions

Install Microsoft's **CMake Tools** and **C/C++** VS Code extensions.
GDB was initially missing from the MinGW installation. Install the matching
debugger from an MSYS2 MinGW x64 terminal:

```bash
pacman -S mingw-w64-x86_64-gdb
```

The debugger should then be available at `C:\msys64\mingw64\bin\gdb.exe`.

### Configure CMake Tools

The following workspace settings in `.vscode/settings.json` reuse the working
build directory and compiler, and configure GDB and the application's environment:

```json
{
		"cmake.useCMakePresets": "never",
		"cmake.buildDirectory": "${workspaceFolder}/build-mingw",
		"cmake.generator": "Ninja Multi-Config",
		"cmake.environment": {
				"PATH": "C:\\msys64\\mingw64\\bin;${env:PATH}"
		},
		"cmake.configureSettings": {
				"CMAKE_C_COMPILER": "C:/msys64/mingw64/bin/gcc.exe",
				"CMAKE_CXX_COMPILER": "C:/msys64/mingw64/bin/g++.exe",
				"CMAKE_CONFIGURATION_TYPES": "Debug;Release"
		},
		"cmake.debugConfig": {
				"MIMode": "gdb",
				"miDebuggerPath": "C:/msys64/mingw64/bin/gdb.exe",
				"args": ["help"],
				"cwd": "${workspaceFolder}",
				"environment": [
						{
								"name": "PATH",
								"value": "C:\\msys64\\mingw64\\bin;$penv{PATH}"
						}
				]
		}
}
```

`cmake.environment` supplies the build environment. The separate
`cmake.debugConfig.environment` supplies the application environment during
debugging. `miDebuggerPath` selects the matching MinGW GDB.

### Fix the startup DLL error

Debugging initially failed before reaching `main()` with `0xc0000135`, meaning
Windows could not find a required DLL. The executable imports these MinGW runtime
DLLs, all of which were present in `C:\msys64\mingw64\bin`:

- `libgcc_s_seh-1.dll`
- `libstdc++-6.dll`
- `libwinpthread-1.dll`

The original debug PATH used `${env:PATH}`. In the installed CMake Tools version
1.24.42, the debug environment is merged before expansion, so this expression
references the PATH value being defined. It retains an unresolved placeholder,
and the extension filters that PATH entry out before launching the debugger.
This behavior was reproduced using the installed extension's expansion and
filtering functions.

Using `$penv{PATH}` in the debug environment avoids that self-reference. For this
launch path, CMake Tools supplies the configure environment as the parent.
The separate `cmake.environment` setting can retain `${env:PATH}`.

Launching VS Code from PowerShell after prepending the MinGW directory also
worked, because VS Code inherited a usable PATH. The corrected workspace setting
avoids needing that workaround. No rebuild is needed for this environment fix.
The GDB architecture warning was not the cause of the startup failure.

### Build Debug and start a session

Use these Command Palette commands:

1. **CMake: Select a Kit**: select GCC from `C:\msys64\mingw64\bin`, not `usr\bin`.
	 Use **CMake: Scan for Kits** first if needed.
2. **CMake: Configure**.
3. **CMake: Select Variant**: select **Debug**.
4. **CMake: Set Debug Target**: select **srs-auditor**.
5. **CMake: Build**.
6. Set a breakpoint on the first statement inside `main()` in
	 `src/srs-auditor/cli/main.cpp`, then run **CMake: Debug**.

Debug builds include debugging symbols and normally disable optimization,
making stepping and variable inspection reliable. This project's Debug executable
is `build-mingw/debug/srs-auditor.exe`.

Start with `"args": ["help"]` to test startup and breakpoints without starting
acquisition. The application requires a command and otherwise exits immediately.
Change this to `"args": ["run"]` for foreground operation; application configuration
and hardware requirements still apply. The help command itself returns exit code
1 in the current implementation, which does not indicate a debugger failure.

Use **F10** to step over, **F11** to step into, and **F5** to continue.
A separate `launch.json` is not required: **CMake: Debug** resolves the executable
from the selected target.
