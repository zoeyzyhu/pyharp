# pyharp audit: one place, and what else can be cut (Zoey)

**Pinned at 2026-11-22T16:57:50Z** (git ls-remote):

- **Audited:** `chengcli/pyharp` main = `35440b2a1b6a8b8e75a4a58b068ff9ffb8ebdc01` (tag v2.8.2).
- **`UCzhangxi/pyharp` main** = `6bc01d5307f074d2ac92b5e484a2442d4deab98d`. This is an ancestor of the audited commit, 4 commits behind (#181 to #184: integrator out tensor, pydisort pins, CI). None of the findings below depend on those 4 commits.

**Scope:**

- **Included:** the C++ core (`src/`), the bindings (`python/csrc/`), the Python package (`python/`) and `data/`.
- **Excluded:** tests and examples, except where they are the only users of something.
- **Method:** findings only, with no build. The two real bugs were reproduced against the installed pyharp 2.8.1 wheel. Their binding lines are unchanged at 35440b2a; see each repro.

Paths are relative to the repository root. `@35440b2a` marks the home at the pinned sha.

## Commands used

| id | command (run in the repo at 35440b2a) |
|----|----|
| C1 | `python3 dupwin.py 6 <files>`: windows of 6 normalised non-trivial lines that repeat across files. C++ set: `git ls-files 'src/*.cpp' 'src/*.hpp' 'src/*.h' 'src/*.cu' 'python/csrc/*' \| grep -v ^src/opacity/mwr/` (130 files, 34 clusters). Python set: `git ls-files 'python/*.py' 'data/*.py' 'examples/*.py'` (36 files, 95 clusters). |
| C2 | AST body-hash of every Python function with 2+ statements (`ast.dump` of the body, sha1), reporting hashes that occur 2+ times. 10 groups. |
| C3 | `git grep -nE '<literal>' -- src python ':!src/opacity/mwr'` for physical constants (8.314, 1.380…e-23, 6.626…e-34, 2.99792458, 5.67…e-8, 2.687…e25, 1.4387…, 101325) |
| C4 | `git grep -nE 'ADD_ARG\([^)]*\)\s*='` against `git grep -nE 'as<[^>]+>\([^)]'`: option defaults in the struct compared with the YAML readers |
| C5 | Header and source reachability: for each `src/**/*.h(pp)`, `git grep` its basename in includes. The CMake globs in `src/CMakeLists.txt:33-41,62-69` are compared with `git ls-files src`. Non-standard extensions come from `git ls-files \| grep -E '\.(cpp_\|cc)$'`. |
| C6 | Per-symbol use count: for each function name declared in a header, `git grep -nw <name> -- src python tests ':!src/opacity/mwr' ':!*.cpp_'` outside its own .hpp/.cpp. 0 means no caller. Every hit was then read by hand; CUDA intrinsics and dispatch registrations were struck from the list. |
| C7 | Python module reachability: each `python/**/*.py`, grepped for `import`/`from`/entry points in `python tests examples setup.py pyproject.toml docs` |
| C8 | Binding vs stub: `python/csrc/*.cpp` `m.def`/`c.attr` names and defaults compared with `python/*.pyi`, then exercised in the installed wheel |

## Counts by kind

| kind | rows | est. lines saved |
|---|---|---|
| dead code: whole files never compiled or imported (D1-D7, D10, D11) | 9 | 66,265 (62,629 of them in `src/opacity/mwr/`) |
| dead code: functions or constants with no caller (D12-D15) | 4 | about 280 |
| copy-paste blocks, C++ (O9-O13; O14 is covered by D11) | 5 | about 185 |
| copy-paste blocks, Python (O15-O19; O20 kept, O21 covered by D7) | 5 | about 95 |
| constants with more than one home (O1-O5; O5b kept) | 5 | 8 (and one source of values) |
| option defaults defined twice (O6-O8) | 3 | 0 (correctness of the single home, not size) |
| unused data and a stub entry point (D8, D9) | 2 | 488 data lines and 10 code lines |
| public module with no in-repo user (D16, owner's call) | 1 | 0 or 454 |
| real bugs (with repro) | 2 | n/a |
| hypotheses | 2 | n/a |

**Estimated total code that can be cut: about 66,840 lines**, plus 488 lines of unused data. Of that, 66,265 is files that nothing builds or imports, and about 570 comes from folding and dead helpers. Excluding `src/opacity/mwr/`, the total is about 4,210 lines (4,660 if D16 is also cut).

## (1) One place: things defined more than once

| # | item | home (path:line@35440b2a) | copies (path:line) | action | est. lines saved | found by |
|---|---|---|---|---|---|---|
| O1 | Boltzmann constant. **The values differ:** C++ uses CODATA-2006 1.3806504e-23, while Python uses exact SI 1.380649e-23 (rel. diff 1.0e-6). | `src/constants.h:7@35440b2a` | `python/spectra/blackbody.py:9`, `python/spectra/hitran_cia_utils.py:15`, `python/spectra/spectrum.py:24` | fold. Choose one value (SI 2019 exact) in `constants.h`, and have the Python modules read `pyharp.constants` (after B1 is fixed). | 3 | C3 |
| O2 | Planck constant. **The values differ:** 6.62606957e-34 vs 6.62607015e-34 (rel. diff 8.8e-8). | `src/constants.h:14@35440b2a` | `python/spectra/blackbody.py:7` | fold as O1 | 1 | C3 |
| O3 | Gas constant. **The values differ:** 8.314462 vs 8.31446261815324. | `src/constants.h:5@35440b2a` | `python/atm_profile.py:24` (`__Constants.kRgas`) | fold as O1 | 1 | C3 |
| O4 | Stefan–Boltzmann constant: the same value in two places | `src/constants.h:18@35440b2a` | `src/radiation/bbflux.cpp:17` (`kSigma`, 5.670374419184429e-8) | fold: `bbflux.cpp` uses `constants::stefanBoltzmann`. The two differ only in digits beyond double rounding of the CODATA value. | 1 | C3 |
| O5 | Speed of light and Loschmidt number | `src/constants.h:16`, `src/constants.h:9@35440b2a` | `python/spectra/blackbody.py:8` (C_LIGHT), `python/spectra/orton_xiz_cia.py:15` (`_LOSCHMIDT_M3`) | fold as O1. The values agree. | 2 | C3 |
| O5b | Second radiation constant, c2 | `src/radiation/bbflux.cpp:16` (1.438776877503933) | `python/spectra/mt_ckd_h2o.py:33` (`radcn2 = 1.4387752`) | **keep.** `radcn2` is MT_CKD's own constant, so changing it would move the MT_CKD reference values. Note it in a comment. | 0 | C3 |
| O6 | Integrator option defaults: `type="rk3"`, `cfl=0.9`, `tlim=1e9`, `nlim=-1`, `ncycle_out=1` | `src/integrator/integrator.hpp:53-57@35440b2a` (ADD_ARG) | `src/integrator/integrator.cpp:26-30` (`node[...].as<T>(literal)`) | fold: `as<T>(op->field())`, as `radiation_band.cpp:136-138` already does for Toon. One home, no drift. | 0 | C4 |
| O7 | Opacity option defaults: `warn_out_of_bounds=false`, `verbose=false`, `nmom=0` | `src/opacity/opacity_options.hpp:59-65@35440b2a` | `src/opacity/opacity_options.cpp:34-35,83` | fold as O6 | 0 | C4 |
| O8 | Radiation band defaults: `verbose=false`, `nstr=4` | `src/radiation/radiation_band.hpp:160,168@35440b2a` | `src/radiation/radiation_band.cpp:69,97` | fold as O6 | 0 | C4 |
| O9 | Opacity `forward` preamble: `ncol/nlyr` from `conc`, `pres`/`temp` required in kwargs, shape checks (about 14 lines) | `src/opacity/helios.cpp:113-127@35440b2a` | `src/opacity/molecule_line.cpp:159-173`, `src/opacity/multiband.cpp:64-78`, `src/opacity/picaso_ck.cpp:61-74`, `src/opacity/respq_table.cpp:160-174`, `src/opacity/molecule_cia.cpp:121` (variant) | fold into one helper in `opacity_options` or a small `opacity_utils.hpp` that returns `{pres, temp}` | 65 | C1 |
| O10 | Constructor check `opacity_files().size() == 1` with the same message and resource lookup (about 6 lines) | `src/opacity/fourcolumn.cpp:16@35440b2a` | `src/opacity/helios.cpp:17`, `src/opacity/molecule_cia.cpp:17`, `src/opacity/molecule_line.cpp:15`, `src/opacity/multiband.cpp:24`, `src/opacity/picaso_ck.cpp:9`, `src/opacity/respq_table.cpp:30` | fold: a `single_opacity_file(options)` helper | 35 | C1 |
| O11 | NetCDF header read (wavenumber, pressure, temperature with the base/delta variants) | `src/opacity/molecule_line.cpp:32-74@35440b2a` | `src/opacity/molecule_cia.cpp:30-72`, plus the close and query blocks `molecule_line.cpp:140-181` against `molecule_cia.cpp:107-140` | fold into `utils/netcdf_opacity_utils` (which already exists) | 60 | C1 |
| O12 | `divide_where_positive` (11 lines, identical) | `src/radiation/band_accumulate.cpp:37@35440b2a` | `src/opacity/water_cloud_temperature.cpp:26` | fold into a single header in `utils/` | 11 | C1 |
| O13 | `layer2level` upper and lower constant-extrapolation and positivity blocks | `src/utils/layer2level.cpp:60-74@35440b2a` | `src/utils/layer2level.cpp:141-155` | fold into a local lambda | 12 | C1 |
| O14 | `read_dimvar_netcdf_double` / `_float` bodies (same code, two element types) | `src/utils/read_dimvar_netcdf.cpp:19@35440b2a` | `src/utils/read_dimvar_netcdf.cpp:49` | **cut instead**: the whole module has no caller (D11) | (in D11) | C1, C6 |
| O15 | CLI helpers `selected_temperatures` / `selected_pressure_bars` (body-identical) | `python/spectra/hitran_molecule_utils.py:700,707@35440b2a` (public) | `python/spectra/atm_overview.py:557,564`, `python/spectra/dump_cli.py:161,168`, `python/spectra/plot_cli.py:103,110` | fold: import from `hitran_molecule_utils` (or move both to `spectra/utils.py`) | 30 | C2 |
| O16 | `_subprocess_worker_entry` (body-identical, 7 lines) | `python/spectra/atm_overview.py:681@35440b2a` | `python/spectra/dump_cli.py:886`, `python/spectra/plot_cli.py:362`, `python/spectra/hitran_molecule_plot.py:581` | fold into `spectra/utils.py`, together with the near-identical `_run_task_via_subprocess` and `_parallel_*_via_subprocess` (`atm_overview.py:709` = `hitran_molecule_plot.py:608`) | 35 | C1, C2 |
| O17 | `_get_help_string` (formatter) and `__call__` (argparse action) | `python/spectra/utils.py:16@35440b2a` | `python/spectra/plot_cli.py:20,30`, `python/spectra/hitran_molecule_plot.py:442` | fold: import from `spectra/utils.py` | 18 | C2 |
| O18 | `_parse_float_list` | `python/spectra/dump_cli.py:91@35440b2a` | `python/spectra/plot_cli.py:41` | fold into `spectra/utils.py` | 8 | C2 |
| O19 | `main_attenuation` / `main_transmission` CLI shims (body-identical) | `python/spectra/hitran_molecule_plot.py:154,184@35440b2a` | `python/spectra/hitran_cia_plot.py:218,246` | fold or keep. They are 3-line entry points, so this is low value. | 6 | C2 |
| O20 | Dataset `__init__` (identical 4-line body) | `python/sonora/sonora.py:108@35440b2a` | `python/h2_cia_legacy/h2_cia_legacy.py:28` | keep. It is too small to share across two packages. | 0 | C2 |
| O21 | Sonora/PICASO legacy grid reader | `python/sonora/get_legacy_data_1460.py:4-290@35440b2a` (the one in use) | `data/optics.py:705-1150` (three copies of the same reader: 1060 grid and 1460 grid, twice) | cut `data/optics.py` (D7) | (in D7) | C1 |

## (2) What else can be cut

| # | item | home (path:line@35440b2a) | evidence | action | est. lines saved | found by |
|---|---|---|---|---|---|---|
| D1 | `src/opacity/mwr/`: the Juno MWR absorbers (19 files, copied from jamrt v1.0.1812131228 per its README) | `src/opacity/mwr/*@35440b2a` | Not in any `file(GLOB …)` in `src/CMakeLists.txt:33-41` (the globs are `opacity/*.cpp`, not recursive) or `:62-69`. No file outside the directory includes `mwr/`, and `git grep -n mwr -- ':!src/opacity/mwr'` finds nothing. Last touched in ba76d26 (2025-04-04). | cut. If the MWR absorbers are wanted, port them as a real opacity module. | 62,629 | C5 |
| D2 | `src/opacity/z.junk/thermodynamics.cc` (canoe code) | `src/opacity/z.junk/thermodynamics.cc@35440b2a` | It is tracked although `.gitignore:41` lists `z.junk/`. Its `.cc` extension matches no glob. | cut | 239 | C5 |
| D3 | `*.cpp_` files: `radiation_band_exchanger.cpp_` (96), `radiation_bands_factory.cpp_` (52), `beer_lambert.cpp_` (58), `write_bin_ascii.cpp_` (54) | `src/radiation/`, `src/rtsolver/`, `src/utils/@35440b2a` | They are never compiled (extension), and their symbols have no callers. | cut | 260 | C5 |
| D4 | `src/rtsolver/rtsolver.hpp` (`BeerLambert` module), whose only implementation is the `.cpp_` in D3 | `src/rtsolver/rtsolver.hpp:1-48@35440b2a` | No includer anywhere | cut | 48 | C5 |
| D5 | `src/opacity/water_cloud.hpp` (`FuWaterLiquidCloud` on canoe's `AirParcel`/`Absorber`) | `src/opacity/water_cloud.hpp:1-106@35440b2a` | No includer. Its includes (`air_parcel.hpp`, `absorber.hpp`) are not in this repo, so it cannot compile. | cut | 106 | C5 |
| D6 | `src/utils/write_bin_ascii.hpp` | `src/utils/write_bin_ascii.hpp:1-14@35440b2a` | Only the `.cpp_` uses it | cut | 14 | C5 |
| D7 | `data/optics.py`: a vendored PICASO `optics.py` | `data/optics.py:1-2785@35440b2a` | Not importable: `from .deq_chem import …` and `from .rayleigh import …` are not in the repo, and it needs bokeh, astropy and numba. No importer or entry point. It raises 6 `SyntaxWarning: invalid escape sequence '\s'` under Python 3.12. | cut | 2,785 | C1, C7 |
| D8 | `data/h2so4.txt` (160), `data/s8_k_fuller.txt` (236), `data/example_hj_profile.csv` (92) | `data/@35440b2a` | No reference anywhere except the dead `data/optics.py` | cut (data, about 104 kB) | 488 data lines | C7 |
| D9 | `fetch-cia-legacy` console script: `main()` is `pass` and imports `requests` and `tqdm` for nothing | `python/api/fetch_cia_legacy.py:1-10@35440b2a`, `pyproject.toml:47` | Installs a command that does nothing | cut (or implement) | 10 | C7 |
| D10 | Superseded `interpn` dispatch: `call_interpn_cpu<N>` / `call_interpn_cuda<N>` and their instantiations | `src/math/math_dispatch.cpp:1-43@35440b2a`, `src/math/interpolation.hpp:25-31` | `interpn` goes through `call_interpn_fused` (`src/math/interpolation.cpp:249,293-294`, `interpn_dispatch.cu:61`). `call_interpn_cpu` is only declared and instantiated, and `call_interpn_cuda` has no definition at all. | cut | 51 | C6 |
| D11 | `src/utils/read_dimvar_netcdf.{hpp,cpp}` | `src/utils/read_dimvar_netcdf.hpp:1-52@35440b2a` | No call to `read_dimvar_netcdf*` anywhere. The two includes (`src/opacity/opacity_options.cpp:9`, `src/radiation/radiation_band.cpp:24`) are unused. | cut the module and the 2 includes | 133 | C6 |
| D12 | `fileio`: 11 of 12 functions have no caller outside the module. `file_exists`, `get_num_cols(_str)`, `get_num_rows(_str)`, `is_blank_line`, `next_line`, `read_data_table` (declared, never defined), `read_data_vector`, `replace_char` and `strip_line` are unused. Only `decomment_file` is used (`src/opacity/fourcolumn.cpp:35`). | `src/utils/fileio.hpp:10-39@35440b2a` | C6 counts (each 0 outside `fileio.*`) | cut to `decomment_file` and the helpers it needs | about 120 | C6 |
| D13 | `element`: `element_names`, `element_symbols`, `element_weights`, `get_atomic_number`, `get_element_name`, `get_element_symbol`, `num_elements_defined` and `num_isotopes_defined` have no caller outside `element.cpp`. Only `get_element_weight` is used (`src/compound.cpp:76`). | `src/element.hpp:11-72@35440b2a` | C6 counts | cut the unused accessors (keep the table) | about 90 | C6 |
| D14 | Small dead helpers: `interp1` (`src/math/interpn.h:85`), `rad2deg` (`src/radiation/parse_radiation_direction.hpp:23`), `format_composition` and `normalize_composition` (`src/compound.hpp:36,56`), `parse_unit_with_default` and `replace_pattern` (`src/utils/parse_yaml_input.cpp:9,38`), `set_default_directories` (`src/utils/find_resource.cpp:71`; its doc mentions an `appinit()` that does not exist) | as listed | C6 counts (0 callers) | cut | about 65 | C6 |
| D15 | Unused constants: `Rgas_cgs`, `kBoltz_cgs`, `hPlanck_cgs`, `cLight_cgs` | `src/constants.h:6,8,15,17@35440b2a` | No use anywhere | cut | 4 | C3, C6 |
| D16 | `python/atm_profile.py` (454 lines, installed as `pyharp.atm_profile`) | `python/atm_profile.py@35440b2a` | No importer in the repo (src, tests, examples, docs). It is a public module, though, so users outside the repo may import it. | **owner's call.** Keep if it is a supported API; otherwise cut, along with O3. | 0 / 454 | C7 |

## Real bugs (reproduced)

### B1. `pyharp.constants.kBoltz` does not exist; the binding misspells it as `kBtolz`

- **Home:** `python/csrc/pyconstants.cpp:14@35440b2a`: `c.attr("kBtolz") = py::float_(constants::kBoltz);`
- **Stub:** `python/constants.pyi:11` declares `kBoltz: float`, so type checkers accept code that then fails at runtime.
- **History:** the line has been unchanged since 8df752f (2026-04-09), which predates v2.2.1, so 2.8.1 and the pinned main both carry it.
- **Repro** (installed wheel, pyharp 2.8.1):

```
$ python -c "import pyharp; from pyharp import constants as c; print(pyharp.__version__); print(hasattr(c,'kBoltz'), hasattr(c,'kBtolz')); c.kBoltz"
2.8.1
False True
AttributeError: module 'pyharp.constants' has no attribute 'kBoltz'
```

- **Fix (one line):** `c.attr("kBoltz") = …`. Keep `kBtolz` as a deprecated alias for one release if anyone depends on it. No caller in this repo uses either name.

### B2. `bbflux_wavenumber(wn1, wn2)` advertises a default `temp: torch.Tensor = 1` that cannot be used

- **Home:** `python/csrc/pyradiation.cpp:20-23@35440b2a`: `py::arg("temp") = 1` on the `(double, double, torch::Tensor)` overload.
- **Stub:** `python/pyharp.pyi:189` declares no default. pybind11 shows `temp: torch.Tensor = 1` in the docstring, but the int default cannot be converted to a Tensor, so the advertised call raises.
- **Repro** (pyharp 2.8.1):

```
$ python -c "import torch,pyharp; print(pyharp.bbflux_wavenumber(1.0, 2.0, torch.tensor([300.0]))); pyharp.bbflux_wavenumber(1.0, 2.0)"
tensor([5.7724e-06])
TypeError: bbflux_wavenumber(): incompatible function arguments. The following argument types are supported:
  ...
    2. bbflux_wavenumber(wn1: typing.SupportsFloat, wn2: typing.SupportsFloat, temp: torch.Tensor = 1) -> torch.Tensor
```

- **Fix:** drop `= 1`, so the argument is required as the stub says.

## Hypotheses (not reproduced)

- **H1:** `python/sonora/get_legacy_data_1460.py:14` (in use) passes `sep='\s+'` in a non-raw string. Today this is a `SyntaxWarning` (Python 3.12). It is expected to become a `SyntaxError` in a future Python, which would break `pyharp.sonora` at import. Fix with `r'\s+'`. Found by `python3 -W always::SyntaxWarning` with `ast.parse` over all `.py` files.
- **H2:** The constant mismatch in O1 to O3 means that Python-side spectra (blackbody, CIA and line units, which use exact SI) and C++ RT (CODATA-2006 `kBoltz`, `hPlanck`) differ at the 1e-6 relative level wherever both are compared or chained. I didn't measure an end-to-end effect.

## Not done

- No build and no test run. The two bugs were reproduced on the installed wheel (2.8.1); the binding lines are byte-identical at 35440b2a.
- Tests and examples were not audited for their own duplication.
- Docs (`docs/`, Doxyfile) were not audited.
- `external/MT_CKD_H2O` (a submodule, used by `python/spectra/mt_ckd_h2o.py:12`) is used, so it is kept.
