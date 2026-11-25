# WLX Isolate

[German documentation](README.md)

A generic Lister wrapper for Total Commander. The actual WLX plugin runs in a
separate process for each viewer, outside Total Commander. If the target plugin
crashes, the wrapper can display an error in the viewer. **Restart plugin** opens
the same file in a new helper process.

The first real plugin tested is CudaLister 1.8.6.3. The wrapper loads, calls and
unloads the target plugin on the same long-lived UI thread in the helper. The
original DLL is never loaded into the TC process, including when querying its
detection string.

## Using the existing CudaLister installation

The project includes a CudaLister configuration example named
`wlx_isolate_cudatext.ini`. Despite its filename, this example targets CudaLister.
Copy it manually to `wlx_isolate.ini` next to the wrapper before using it.
Its Win64 target is:

`C:\Program Files\totalcmd\plugins\wlx\CudaLister\cudalister.wlx64`

It passes `C:\Program Files\totalcmd\lsplugin.ini` as the existing settings file
and preserves the detection filter for txt, ini, yaml, readme, list, md and pas.
The example selects German UI text; use `Language=en` for English.

**Manually select the wrapper in Total Commander instead of the existing
CudaLister entry.** Keep these Win64 files together in one directory:

- `wlx_isolate.wlx64`
- `wlx_isolate_host64.exe`
- `wlx_isolate.ini`

The wrapper can also be used directly from the project directory once its
runtime INI has been prepared. With a normal `.wlx` registration, TC selects the
adjacent `.wlx64` version when running in 64-bit mode.

The 32-bit version uses `wlx_isolate.wlx` and `wlx_isolate_host.exe`. It requires
an actual 32-bit DLL of the target plugin. Only the 64-bit CudaLister DLL was
present in the installed CudaLister directory when the example was prepared.

No installed plugins, TC settings or archives have been changed by this project.
Do not rename an original plugin DLL to the wrapper's name or overwrite it with
the wrapper.

## Original directory and settings

`Plugin`, `Plugin32` and `Plugin64` point to the original plugin file. Before
loading it, the helper changes its working directory to the original plugin's
directory and uses `LoadLibraryExW(..., LOAD_WITH_ALTERED_SEARCH_PATH)`. The target
plugin's module path remains its actual path. Adjacent lexers, themes, DLLs and
other resources stay in their original directory; they do not need to be copied
next to the wrapper.

A `<target-plugin>.ini` next to the original DLL takes precedence. Otherwise,
the wrapper passes `BackendIni`, or, if it is empty, the INI path supplied by TC
through `ListSetDefaultParams`. The wrapper does not write these INIs. The target
plugin may save its usual settings there.

The WLX ABI limits the forwarded INI path to an ANSI field of `MAX_PATH` bytes.
Paths that are too long or cannot be represented without conversion loss are
rejected.

## Configuration

For other plugins, manually copy `wlx_isolate_default.ini` to `wlx_isolate.ini`
and set the target plugin path. The default file is a reference template and is
never read at runtime. If the wrapper DLL is renamed, its INI must use the same
new base name. The helper executable names stay unchanged.

| Key in `[Wrapper]` | Default | Meaning |
|---|---|---|
| `Plugin` | empty | Original WLX file. Relative paths are resolved against the wrapper directory. Environment variables are expanded. On Win64, an existing `.wlx64` file is preferred when the configured path ends in `.wlx`. |
| `Plugin32` | empty | Explicit Win32 target; otherwise uses `Plugin`. |
| `Plugin64` | empty | Explicit Win64 target; otherwise uses `Plugin`. |
| `BackendIni` | empty | Fallback settings file for the target plugin, as described above. |
| `DetectString` | empty | Explicit TC detection string. If empty, queries the original plugin in a short-lived isolated helper. |
| `FallbackOnReject` | 1 | Requests TC's internal text mode (key `1`) when the original plugin normally rejects the first file. `0` keeps the rejection message for diagnosis. |
| `StartupTimeoutMs` | 15000 | Time limit for initialization and asynchronous file loading/reloading. |
| `CommandTimeoutMs` | 3000 | Time limit for commands and searches. Visible target-plugin dialogs may wait for user input. |
| `ShutdownTimeoutMs` | 3000 | Waiting interval during shutdown. If the plugin does not respond and has no visible original dialog, the user can keep waiting or stop the helper. |
| `Language` | auto | `auto`, `de` or `en` for the wrapper UI. |

Timeouts accept values from 100 to 300000 ms. Invalid values use the default.
Whitespace-delimited inline comments starting with `;` or `#` are removed
outside quotes. Quoted values and URL fragments are preserved.

## Supported WLX features

- ANSI and Unicode loading, reloading, closing and detection strings.
- `ListSetDefaultParams`, `ListSendCommand`, text search, search dialogs and
  printing.
- ANSI fallback when the original plugin only provides ANSI exports. Text that
  cannot be represented in the current Windows code page is not silently
  replaced.
- Native window embedding, resizing, Quick View focus handling, forwarded keys
  and standard Lister status messages.
- A separate F3 Lister opened from the foreground is activated after
  asynchronous loading, with keyboard focus in the original editor. Quick View
  leaves focus in the TC panel until the user explicitly enters the preview.
  Switching to another window during loading is respected.
- TC dark-mode parameters are handled and forwarded through `lc_newparams`.
  The original plugin controls the appearance of its own UI.

Opening and reloading files is asynchronous: the wrapper returns a child window
early. If the original plugin subsequently rejects the first file normally,
the helper is closed and exactly one mode key `1` is posted to the supplied
Lister parent. This requests TC's internal text mode without reopening the
target plugin. No focus change is requested for Quick View. Holding Ctrl, Alt
or Shift delays the mode message until the modifier keys are released.

Crashes, timeouts and technical startup errors remain visible for diagnosis.
With `FallbackOnReject=0`, ordinary rejection messages remain visible too.
If reloading is rejected, the previous editor is preserved and a message appears
above it, protecting its contents and any unsaved changes. A missing reload
export is reported immediately using the normal WLX error return value.

In CudaLister, "The plugin could not display this file" can indicate normal
rejection of binary contents. A compiled `.o` file is not a text file; forcing
plugin display does not bypass the original plugin's content check. The wrapper
requests text mode in this case. For binary data, TC's hex mode can then be
selected. To view source code, open the corresponding `.pas`, `.pp` or `.lpr`
file. Restarting the helper does not make CudaLister accept binary data.

**Limitations:** Compatibility is not guaranteed for every WLX plugin.
Cross-process windows, modal dialogs, focus and special assumptions about the
host need practical testing for each plugin. Thumbnails
(`ListGetPreviewBitmap*`) and a direct TC export forwarding arbitrary
`WM_NOTIFY` structures are not implemented. Native notifications at the helper
parent can be forwarded to the original plugin within the helper process.
Plugins with additional requirements need a separate extension.

Windows may couple input queues when windows from different processes are
embedded. Bounded IPC waits therefore do not guarantee complete independence
from every possible GUI or driver hang.

Crash isolation is not a security sandbox for malicious plugins. A helper crash
or forced termination loses unsaved changes in that helper. Use test copies
only when testing editing and saving.

## Lifetime and cleanup

Each viewer owns its mapping and event handles. Only an explicit list of handles
is inherited by the helper. Its handle to the host process grants only
`SYNCHRONIZE` access. A job object with `KILL_ON_JOB_CLOSE` prevents orphaned
helpers when the host exits.

Normal shutdown calls the original close export and `FreeLibrary` on the
helper's UI thread. Crash and timeout paths stop only the affected helper.
The wrapper DLL has no worker threads or window procedures left alive after
closing. Its window class is unregistered after the last viewer closes.
DLL detach contains only a short fallback for unregistering an unused class.

## Building and testing

Toolchain: the available Zig 0.16.0 (`zig cc`) and the Windows API. The wrapper DLL
uses neither LCL nor the Free Pascal runtime. This C project is therefore built
with the build script rather than `lazbuild`. All caches and build outputs stay
inside the project directory.

Always update the wrapper and its helper together. Their internal protocol
versions must match (currently version 2).

```powershell
.\build.ps1 -Target all -Tests
python test\run_tests.py --bits all
python test\run_tests.py --bits all --real
python test\run_tests.py --bits all --variant ansi
python test\run_tests.py --bits all --variant no-next
python test\check_orphans.py
```

Real-plugin tests use only DLL copies from the existing reference archive under
`analysis_cudalister`, copied resources and test INIs. No real user documents are
edited. Logs are written to `results.log` in `test/run32`, `test/run64`,
`test/real32` and `test/real64`. Additional variants use directories with the
`-ansi` and `-no-next` suffixes.

SHA-256 mappings are recorded in `test/results.json`, `test/results-ansi.json`,
`test/results-no-next.json` and `test/real-results.json`. Final artifact hashes
and the verified target/profile matrix are in `build/verified-artifacts.json`.

Both Win32 and Win64 passed the native standard, ANSI, missing-reload-export and
real CudaLister test suites. The fallback tests verify a single mode message,
Quick View focus preservation, and safe viewer closure and immediate DLL unload
by the receiving native host. **The resulting mode switch in an actual TC
session has not yet been verified.** Native host tests do not prove complete
compatibility or the absence of all resource leaks.

See [STATUS.md](STATUS.md) for the current verification record and remaining
limitations.
