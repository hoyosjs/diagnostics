# DBI ↔ DAC PAL decoupling — analysis & checklist

## Problem
On Unix/macOS, `mscordbi` (DBI) ships **without its own PAL**. It borrows the PAL
compiled into `mscordaccore` (DAC): `mscordbi/CMakeLists.txt` links `mscordaccore`
("share the PAL in the dac module") and resolves DBI's `PAL_*` imports to the DAC's
`DAC_`-prefixed exports via `palredefines.S`. Result: exactly one PAL instance per
process, living in the DAC.

Goal: let each module have its own PAL (or DBI need no PAL), enabled by the cDAC /
contract-descriptor work. Constraint: **this release must still support DBI loading
the legacy (regular) DAC**, which shares the PAL.

## What is NOT actually coupling (verified)
- **Signals:** DAC and DBI init with `PAL_INITIALIZE_DLL == PAL_INITIALIZE_NONE (0)`.
  Hardware handler registration (`SIGSEGV/SIGBUS/SIGILL/SIGFPE/SIGTRAP`, macOS Mach
  ports) is gated on `PAL_INITIALIZE_REGISTER_SIGNALS`, which is never set. No handler
  conflict between two PALs. (`pal.h:165`, `signal.cpp`, `machexception.cpp:1397`)
  - Only residual global effect: `signal(SIGPIPE, SIG_IGN)` runs unconditionally —
    idempotent across two PALs.
- **Forward exceptions (DAC→DBI):** HRESULT-only by contract via `EX_CATCH_HRESULT`
  (`dacdbiimpl.cpp`). On Unix `PAL_CPP_TRY/THROW` is native C++ EH through the system
  unwinder, not the PAL (`pal.h:3782-3790`). PAL SEH (`__try/__except`) is Windows-only.
- **DacDbi contract carries no PAL objects:** interface is blittable/stateless/
  marshalable — TIDs not handles; no PAL `HANDLE`/`HMODULE`/TLS crosses. DAC entry uses
  `g_dacMutex` = `minipal_mutex` (PAL-independent). Heaps already separate (`IAllocator`
  + statically-linked CRT). `GetThreadHandle` is `#ifdef TARGET_WINDOWS` only.
- **DBI's PAL usage is DBI-local:** RS event thread, Ctrl-C thread, sync events
  (`m_leftSideEventAvailable`, `m_stopWaitEvent`, `m_markAttachPendingEvent`…),
  `WaitForMultipleObjectsEx`, `WszLoadLibrary`/`GetProcAddress`, `GetLastError`, `wcs*`.
  All created and consumed inside DBI — needs *a* PAL, not the *same* PAL as DAC.

## What genuinely couples them
1. **Build-time linkage:** DBI links no full PAL (only `debug-pal` = twowaypipe);
   imports satisfied by the DAC's exported PAL.
2. **Two throwing reverse callbacks (DBI→DAC), native C++ exceptions:**
   - `IAllocator::Alloc` — DBI does `new BYTE[lenBytes]; // throws`; DAC calls it inside
     `EX_TRY` (`process.cpp`, `dacdbiimpl.cpp:104`).
   - `IMetaDataLookup::LookupMetaData` — "throws on exceptional circumstances"; DAC wraps
     in `EX_TRY/EX_RETHROW` (`dacdbiimpl.cpp:442`).
   PAL-independent on Unix, but they are the real cross-DSO exception edges (cross-DSO
   RTTI/ABI exposure).
3. **The dbgshim/SOS legacy `OpenVirtualProcess*` handle path (the one shared-PAL path):**
   dbgshim has its own PAL (`coreclrpal`). A PAL `HMODULE` is a PAL-private `MODSTRUCT*`
   validated against *that* PAL's module list. Legacy `OpenVirtualProcessImpl` takes an
   `HMODULE` that DBI feeds to its (DAC-shared) PAL's `GetProcAddress`, so dbgshim must
   recreate the DAC handle in the DAC's PAL first:
   ```cpp
   // diagnostics/src/dbgshim/debugshim.cpp (also SOS runtimeimpl.cpp:744)
   LoadLibraryWFnPtr loadLibraryWFn = (LoadLibraryWFnPtr)GetProcAddress(hDac, "LoadLibraryW");
   hDac = loadLibraryWFn != NULL ? loadLibraryWFn(dacModulePath) : NULL;
   ```
   `OpenVirtualProcessImpl2` instead takes `pDacModulePath` (a path) and DBI loads the DAC
   itself (`WszLoadLibrary`), so the handle is born in DBI's PAL — inherently compatible
   with a DBI-private PAL.

## Decoupling checklist

### Stage 0 — Ship a parallel `mscordbi_universal` (de-risk the product)
0. **Build a second DBI DLL, `mscordbi_universal`, that links its own PAL**, leaving the
   existing `mscordbi` (shared-DAC-PAL) untouched and shipping. This isolates all the risk
   of PAL separation behind a new binary that debuggers can opt into, so the in-market
   DBI + legacy DAC path keeps working unchanged this release.
   - New `dlls/mscordbi_universal/CMakeLists.txt` reusing the `cordbdi` static lib and DBI
     sources, but **link a private PAL** (`coreclrpal` + `palrt`) instead of `mscordaccore`.
     Do **not** add the `palredefines.S` / `DAC_`-prefixed import mapping.
   - **Keep it out of the shared framework.** Unlike `mscordbi`
     (`install_clr(TARGETS mscordbi DESTINATIONS . sharedFramework COMPONENT debug)`), do
     **not** install `mscordbi_universal` to `sharedFramework` — it ships only through the
     cDAC transport package, mirroring how `mscordaccore_universal` is packaged rather than
     placed in the framework. Install it to a non-framework destination (e.g. `.` only) so
     it never lands in the runtime pack / shared framework layout.
   - Keep the same exported entrypoints; standardize this binary on `OpenVirtualProcessImpl2`
     (path-based DAC load) so its DAC `HMODULE` is created in its own PAL.
   - Have dbgshim/SOS resolve `mscordbi_universal` when the universal/cDAC contract path is
     selected, falling back to `mscordbi` otherwise. Gate selection so the legacy path is
     the default until the universal binary is proven.
   - **Ship it via the cDAC transport package.** Add `mscordbi_universal` as a `NativeBinary`
     in `src/installer/pkg/projects/Microsoft.DotNet.Cdac.Transport/Microsoft.DotNet.Cdac.Transport.pkgproj`
     alongside the existing `mscordaccore_universal` entry, so diagnostics tooling picks up the
     universal DBI from the same transport package (symbols auto-discovered as with the DAC).
   - Once `mscordbi_universal` is validated, converge: either promote it to be the only DBI
     or fold its PAL-owning build config back into `mscordbi` and delete the shared-PAL
     variant. Stages 1–6 below describe that converged end state.

### Converged end state (after `mscordbi_universal` is proven)
1. **Give DBI its own PAL** (or replace its handful of PAL facilities: events, threads,
   `WszLoadLibrary`, waits, `wcs*`). Stop borrowing the DAC's PAL in
   `mscordbi/CMakeLists.txt` + `palredefines.S`.

2. **Standardize on `OpenVirtualProcessImpl2` and remove the original `OpenVirtualProcessImpl`
   export path.** `OpenVirtualProcessImpl2` is always exported by mscordbi and just loads
   the DAC itself (via `WszLoadLibrary`) before delegating — so the DAC `HMODULE` is created
   in whichever PAL DBI uses. Because we control the DBI, there is no need to keep the
   legacy `HMODULE`-taking entrypoint or the dbgshim/SOS `useLegacyOvp` handle-recreation
   branch. Drop:
   - the `OpenVirtualProcessImpl` (HMODULE) entrypoint and the deprecated `OpenVirtualProcess`
     overloads in `runtime` `process.cpp`,
   - the `useLegacyOvp` fallback + DAC-PAL `LoadLibraryW` recreation in
     `diagnostics/src/dbgshim/debugshim.cpp` and `src/SOS/Strike/platform/runtimeimpl.cpp`.
   This eliminates the single genuine shared-PAL path.

3. **Never pass a PAL `HANDLE`/`HMODULE` across the two independent PAL instances.**
   Keep the DacDbi/data-target boundary blittable (TIDs, addresses, HRESULTs only).

4. **Neutralize the two throwing reverse callbacks** so no C++ exception crosses the
   DBI→DAC DSO boundary: make `IAllocator::Alloc` and `IMetaDataLookup::LookupMetaData`
   nonthrowing (return null/HRESULT) or add DBI-side catch/translate adapters.

5. **No signal-owner conflict to resolve.** Neither PAL installs hardware handlers
   (flags = `PAL_INITIALIZE_NONE`); `SIGPIPE`-ignore is idempotent. Keep DAC init
   independent; confirm the DAC still initializes its own PAL on load (`DllMain2`).

6. **PAL init / module registration becomes self-contained per module.** With a DBI-private
   PAL, `PAL_InitializeDLL`/`PAL_RegisterModule`/`init_count` are per-instance bookkeeping,
   not cross-module coupling.

## Follow-up cleanup (after the shared-PAL path is removed)
7. **Unify the two DBI binaries and remove the now-unused PAL redefines infrastructure.**
   The `mscordbi` / `mscordbi_universal` split exists only to de-risk the transition. Once
   the universal (private-PAL) DBI is proven, converge to a **single DBI DLL** — keep the
   `mscordbi` name/identity but with `mscordbi_universal`'s private-PAL build config — and
   delete the parallel variant so there is exactly one DBI shipping.
   - Fold the private PAL link (`coreclrpal` + `palrt`) into `dlls/mscordbi/CMakeLists.txt`;
     delete `dlls/mscordbi_universal/` and its build wiring.
   - Update dbgshim/SOS to resolve the single `mscordbi` again (drop the universal-vs-legacy
     selection logic).
   - Update the cDAC transport package
     (`Microsoft.DotNet.Cdac.Transport.pkgproj`): drop the `mscordbi_universal` `NativeBinary`
     entry once the unified `mscordbi` is what ships (adjust to reference `mscordbi` if the
     transport package should still carry the DBI, or remove it if the framework copy now
     suffices). Ensure the unified `mscordbi` re-enters the shared framework install
     (`DESTINATIONS . sharedFramework`) that Stage 0 deliberately withheld from the universal
     variant.

   With no consumer borrowing the DAC's PAL, the `DAC_`-prefixed PAL export +
   import-redirection machinery is dead. Delete/retire:
   - `dlls/mscordac/palredefines.S` and the `PAL_REDEFINES_FILE` wiring in
     `src/coreclr/CMakeLists.txt`.
   - The `palredefines.inc` generation (`generateredefinesfile.sh` / `.ps1`,
     `pal_redefines_file` target, `-prefix2 DAC_` invocation) in
     `dlls/mscordac/CMakeLists.txt`, and the `PAL_REDEFINES_FILE` consumption +
     `pal_redefines_file` dependency in `dlls/mscordbi/CMakeLists.txt`.
   - The `mscordaccore` link dependency ("share the PAL in the dac module") from the DBI
     build.
   - If the DAC no longer needs to re-export PAL under the `DAC_` prefix for any other
     consumer, also retire `generate_exports_file_prefix(... DAC_)` / `libredefines.S`
     (`LIB_REDEFINES_INC`, `lib_redefines_inc`) in `dlls/mscordac/CMakeLists.txt`.
   - Verify no remaining references to `DAC_`-prefixed PAL symbols across runtime +
     diagnostics before deleting the generators.
