# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an IDA Pro plugin that demonstrates how to extend IDAPython from native C++ plugins using pybind11. The plugin piggybacks on IDAPython's existing Python interpreter (does NOT create a separate interpreter) and injects C++ functions and classes into the `idaapi.ext` namespace.

## Build System

**Dependencies are managed via vcpkg manifest mode** (`vcpkg.json`). Dependencies install automatically during CMake configuration.

### Building

```bash
# Configure (vcpkg toolchain required)
cmake -B build -DCMAKE_TOOLCHAIN_FILE=%VCPKG_ROOT%\scripts\buildsystems\vcpkg.cmake

# Build
cmake --build build --config Release
```

Output DLL is automatically copied to `%IDASDK%\bin\plugins\python_ext.dll` (or `python_ext64.dll` for 64-bit).

### Requirements

- **IDASDK environment variable** must be set to the IDA SDK path
- **Windows only** (tested on Windows with Python 3.11)
- **Release builds only** - Debug builds are explicitly blocked in `idasdk.h:24`
- **vcpkg** must be installed with `VCPKG_ROOT` environment variable set

## Architecture

### Critical Timing & Lifecycle

1. **IDA loads database** → triggers `ui_initing_database` event
2. **IDAPython initializes** → creates the one and only Python interpreter in the process
3. **This plugin receives `ui_initing_database`** → immediately:
   - Pins itself in memory (platform-specific: `LoadLibraryA()` on Windows, `dlopen()` on Unix) to prevent unloading
   - Calls `register_extensions()` to inject C++ bindings into `idaapi.ext`

**Key invariant**: Never create a new interpreter. Always use `py::gil_scoped_acquire` to access the existing one.

### Extension Registration (`extension.cpp`)

All C++ bindings are registered via the `EXTENSION_TABLE` vector of lambdas. Each lambda receives a `py::object&` (the `idaapi.ext` object) and adds functions/classes to it:

```cpp
static std::vector<std::function<void(py::object&)>> EXTENSION_TABLE = {
    [](auto& obj) {
        obj.attr("my_print") = py::cpp_function(&my_print);
    },
    [](auto& obj) {
        py::class_<MyUtil>(obj, "MyUtil")
            .def(py::init<const std::string&>(), py::arg("name") = "no name")
            .def_readwrite("name", &MyUtil::name_)
            // ...
    }
};
```

To add new extensions:
1. Define the C++ function/class in `extension.cpp`
2. Add a lambda to `EXTENSION_TABLE` that binds it to the `ext` object
3. The extension becomes available in Python as `idaapi.ext.<name>`

### Plugin Entry Point (`plugin.cpp`)

The `run()` method (triggered by Ctrl-4) demonstrates:
- Calling Python from C++ (requires `py::gil_scoped_acquire`)
- Importing Python modules (`py::module_::import()`)
- Calling Python functions and methods
- Type conversions between C++ and Python

### Important Files

- **`idasdk.h`**: Common includes for pybind11 and IDA SDK. Disables MSVC warnings and blocks Debug builds.
- **`extension.cpp`**: Extension registration system. Add new C++ bindings here.
- **`plugin.cpp`**: Plugin lifecycle and demo code showing bidirectional Python ↔ C++ interaction.
- **`mymodule.py`**: Example Python module used by the demo.

## Common Patterns

### Calling Python from C++

```cpp
py::gil_scoped_acquire acquire;  // Always acquire GIL first
py::module_ os = py::module_::import("os");
py::object getcwd = os.attr("getcwd");
std::string cwd = getcwd().cast<std::string>();
```

### Registering C++ to Python

Add to `EXTENSION_TABLE` in `extension.cpp`:

```cpp
[](auto& obj) {
    obj.attr("my_function") = py::cpp_function(&my_function);
}
```

Then use from Python:
```python
import idaapi
idaapi.ext.my_function()
```

## Platform Notes

- **Windows**: Uses `LoadLibraryA("python_ext.dll")` to pin plugin in memory
- **macOS**: Uses `dlopen("python_ext.dylib")` to pin plugin in memory
- **Linux**: Uses `dlopen("python_ext.so")` to pin plugin in memory
- **Python version**: Tested only with Python 3.11
- **IDA version**: Requires IDA 9.0+ (always EA64, no 32-bit support)

## Key Constraints

- **No Debug builds**: Pybind11 release/debug mismatch causes crashes. Use `RelWithDebInfo` or `Release`.
- **No separate interpreter**: This plugin shares IDAPython's interpreter. Never call `py::scoped_interpreter`.
- **Always acquire GIL**: All Python operations from C++ must be wrapped in `py::gil_scoped_acquire`.
- **Plugin must stay loaded**: Self-pinning prevents crashes from dangling Python references to C++ objects.
