# Contribution 3: Make `--find-interpreter` require opt-in to prereleases

**Contribution Number:** 3  
**Student:** Tien Nguyen (GitHub: [@tientnc](https://github.com/tientnc))  
**Issue:** [PyO3/maturin #1975](https://github.com/PyO3/maturin/issues/1975) (Fork: not yet created)  
**Status:** Phase III in progress (fix implemented and tested locally on a feature branch; not yet forked/pushed or submitted as a PR)

---

## Why I Chose This Issue

I chose `PyO3/maturin` issue #1975 to get experience outside the Swift/Java ecosystem from my first two contributions. `maturin` is a mature Rust project (5,600+ stars) that builds Python wheels for Rust extensions, and picking it let me learn Rust CLI conventions along with Python's `sysconfig`/`sys.version_info` internals.

Before selecting it, I checked whether it was actually claimable: open, labeled `good first issue`, and unassigned. There was some history to work through, though. `fatelei` commented "i want to have a try" back in March 2024 and got assigned by maintainer `messense`, but maintainer `konstin` unassigned `fatelei` on 2026-01-05 (I confirmed this through the GitHub Timeline API, which turned out to be more reliable here than the `/events` REST endpoint), and no PR ever came out of it (`search/issues?q=repo:PyO3/maturin+1975+type:pr` returns 0 results). I also chased down a cross-reference that looked like competing work at first: a bot review comment on an unrelated repo's PR (`CelestoAI/SmolVM#146`) mentioned the issue in passing, but it wasn't actual work on the fix. Since a maintainer had deliberately freed the issue up, I treated it as genuinely available.

I posted a comment on the issue describing my reproduction and plan, but haven't heard back from a maintainer yet.

---

## Understanding the Issue

### Problem Description

`maturin build`/`publish` with `--find-interpreter` (`-f`) auto-discovers every installed Python interpreter matching the supported minor-version range and builds a wheel for each one it finds, with no distinction between a final release and a prerelease (alpha/beta/rc) build. Maintainer `davidhewitt` opened the issue as a follow-up to a related discussion, noting this risks users accidentally publishing wheels for prerelease Python versions to PyPI before they meant to support them in CI.

### Expected Behavior

`--find-interpreter` should skip any discovered interpreter whose `sys.version_info.releaselevel` is not `"final"`, unless the user opts in with a new `--allow-prereleases` flag. When a prerelease interpreter is found and skipped, maturin should print a message telling the user it exists and hinting at `--allow-prereleases`, rather than silently ignoring it.

### Current Behavior

There is no prerelease filtering anywhere in the discovery path, and no `--allow-prereleases` flag exists in the CLI at all.

### Affected Components

- `src/python_interpreter/get_interpreter_metadata.py`: the script maturin runs inside each candidate interpreter to collect metadata. It never reads `sys.version_info.releaselevel`/`serial`.
- `src/python_interpreter/discovery.rs`: `InterpreterMetadataMessage` (the struct populated from that script's JSON output) has no `releaselevel` field. `from_metadata_message()` builds a `PythonInterpreter` with no maturity check, and `cpython_candidate_names()`/`find_all()` generate and probe candidate binaries (e.g. `python3.14`) purely by minor version.
- `src/build_options.rs`: `PythonOptions` has no `allow_prereleases` field or CLI flag.
- `src/cross_compile.rs`: worth flagging so it isn't mistaken for existing support. `releaselevel` does appear in this file, but only inside PEP 739 `build-details.json` fixtures used by tests. The struct that actually parses those files (`BuildDetailsImplementation`) declares only a `name` field, and serde silently drops unknown keys, so `releaselevel` is never read even on the cross-compile path. In other words, nothing anywhere in maturin currently consumes this field.

---

## Reproduction Process

### Environment Setup

My machine didn't have Rust installed, so I installed it with `rustup`, the standard Rust installer. No `sudo` was needed. I then cloned the maturin repo and built it with `cargo build`, no extra setup required. I haven't made a fork yet since I'm still in the investigation stage, not writing code.

### Steps to Reproduce

Since the maintainer hadn't replied yet, I wanted a simple way to double-check the bug is still there before waiting further. Here's what I did, in plain terms:

1. **Check what Python itself already knows.** Run `python3 -c "import sys; print(sys.version_info)"`. Python prints a `releaselevel` value (e.g. `final`), so Python already tracks whether it's a real release or a preview build like alpha/beta/rc.
2. **Run maturin's own script that's supposed to collect this info.** `python3 src/python_interpreter/get_interpreter_metadata.py`. Its output is missing the `releaselevel` field completely, even though Python hands it over for free in step 1.
3. **Search maturin's code for anything using that field.** `grep -rn "releaselevel" src/`. It only shows up in one unrelated test file for a different feature (cross-compiling to other operating systems), never in the actual interpreter-discovery code that `--find-interpreter` uses.
4. **Check if the flag from the issue exists.** Run `maturin build --help` and look for `--allow-prereleases`. It's not there.

**Result:** the feature described in the issue simply hasn't been built yet. It's not a partial fix and not a bug that broke it later, the code for it doesn't exist.

I repeated all four steps again on 2026-07-14 after pulling the newest changes from the project (8 new commits since my first check), to make sure nothing had changed while waiting for a reply. The result was identical: still no `releaselevel` field, still no `--allow-prereleases` flag.

### Reproduction Evidence

- **Commit showing reproduction:** none yet. I haven't forked or branched since this is Phase II (investigation), not Phase III (implementation).
- **Logs (first check, commit `e54b7775`, and repeated on commit `8f981619`, same result both times):**
  ```
  $ python3 -c "import sys; print(sys.version_info)"
  sys.version_info(major=3, minor=12, micro=3, releaselevel='final', serial=0)

  $ python3 src/python_interpreter/get_interpreter_metadata.py
  {"implementation_name": "cpython", "major": 3, "minor": 12, "abiflags": "",
   "interpreter": "cpython", "platform": "linux-x86_64", "system": "linux", ...}
  # no "releaselevel" key anywhere in this output

  $ ./target/debug/maturin build --help | grep -i prerelease
  # (no output, the flag doesn't exist)
  ```
- **My findings:** the missing piece isn't a filter condition hiding somewhere in the Rust code. The information is dropped at the very first step, the small Python script maturin runs to inspect each interpreter, so nothing later in the program could use it even if it wanted to. That's the actual starting point for the fix: add the field where it's collected, before anything downstream can act on it.

---

## Solution Approach

### Analysis

Every Python install already knows whether it's a finished release or an early preview (alpha/beta/rc), the same way software might be labeled "stable" or "beta." maturin just never asks Python for that label. When it scans a computer for available Python versions, it runs a small helper script that collects basic details like the version number, but that script skips the one field that would say "this is a preview build." Because that detail is never collected in the first place, nothing later in maturin has any way to know an installation is a prerelease, so it treats every version the same and builds a wheel for it regardless.

More technically: the probe script (`get_interpreter_metadata.py`) never reads Python's `sys.version_info.releaselevel`, so the `InterpreterMetadataMessage` struct that carries this data into the Rust code has no field for it. Every function downstream of that, like `from_metadata_message()` and `find_all()`, has nothing to check even if it wanted to. The fix has to start at that first collection point, not somewhere deeper in the discovery logic.

### Proposed Solution

Fix it in the same order the information flows, starting at the point where it's currently lost. Four connected changes:

1. **Collect the label.** Add `releaselevel` to the small probe script (`get_interpreter_metadata.py`) so every interpreter maturin inspects now reports whether it's a `final` release or a preview (`alpha`/`beta`/`candidate`).
2. **Carry it into the Rust side.** Add a matching `releaselevel` field to `InterpreterMetadataMessage`, and record on each discovered interpreter whether it's a prerelease.
3. **Filter at the right moment.** In the interpreter resolver — the one place that already knows both which interpreters were found *and* which flags the user passed — drop any auto-discovered prerelease and print a short notice (e.g. "Skipping prerelease Python 3.14.0a1; pass `--allow-prereleases` to build for it"). Interpreters the user names explicitly with `-i` are left untouched, since naming one by hand is already an opt-in.
4. **Add the opt-in.** A new `--allow-prereleases` flag that turns the filter off.

Keeping the decision in the resolver rather than deeper in discovery means the skip-or-keep logic and its warning have all the context they need in one place, and the lower-level discovery code stays a plain "report what exists" layer.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** `-f/--find-interpreter` builds a wheel for every Python it finds, preview builds included, with no way to say "released versions only." I need to add prerelease detection and make prereleases opt-in via `--allow-prereleases`, without changing behavior for interpreters the user names explicitly.

**Match:** `gil_disabled` is a near-identical, recent precedent — a boolean that starts in `get_interpreter_metadata.py`, rides through `InterpreterMetadataMessage`, and is consumed on the Rust side (`abiflags.rs`, `config.rs`). I'll follow that same path for `releaselevel`. For the "skip an interpreter and say why" behavior, `from_metadata_message()` already does exactly this for outdated versions and Windows architecture mismatches (returns `Ok(None)` and prints a `⚠️` (or similar) notice), so the skip pattern and message style are already there to copy. (`releaselevel` also appears in `cross_compile.rs` test fixtures for PEP 739 `build-details.json`, but the parser there doesn't actually read the field, so that path is separate and out of scope.)

**Plan:** Step-by-step implementation plan
1. `get_interpreter_metadata.py`: add `"releaselevel": sys.version_info.releaselevel` to the metadata dict.
2. `discovery.rs`: add `releaselevel: String` to `InterpreterMetadataMessage`; in `from_metadata_message()`, compute `is_prerelease` (`releaselevel != "final"`) and store it on the returned `PythonInterpreter`. Bundled/sysconfig interpreters (which never run the script) default to not-prerelease.
3. `config.rs`: add the `is_prerelease` field to `PythonInterpreter`/`InterpreterConfig`.
4. `build_options.rs`: add `--allow-prereleases` (`allow_prereleases: bool`) to `PythonOptions`, alongside `find_interpreter`.
5. `resolver.rs`: thread `allow_prereleases` into `InterpreterResolver`; in `discover_native()`'s `find_interpreter` branch, filter out prereleases unless the flag is set, printing a per-interpreter notice for each one skipped.
6. Tests: unit test that `alpha`/`beta`/`rc` flag as prerelease and `final` does not; resolver test that the filter drops prereleases when the flag is off and keeps them when on.

**Implement:** done locally on branch `allow-prereleases`. The change follows the plan exactly and stays small (89 insertions / 3 deletions across 7 files, most of it a one-line `is_prerelease` field default repeated across constructors). Key decision: I put `is_prerelease` on `PythonInterpreter` (a runtime-discovered property) rather than on `InterpreterConfig`, because `InterpreterConfig` is deserialized from bundled sysconfig data and constructed in ~15 places — adding a field there would be a large, sprawling diff and semantically wrong (bundled configs are always released versions). I also kept the filter in the resolver rather than in `find_all`, so `maturin list-python` (which also calls `find_all`) still shows every interpreter. Fork/PR links will be added when submitted.

**Review:** before opening a PR, check against maturin's contribution guidelines — run `cargo fmt`, `cargo clippy`, and `cargo test`; keep the diff minimal and matched to each file's style; make sure the new flag's `#[arg]` help text is clear.

**Evaluate:** build maturin and run `--find-interpreter` on a machine with both a final and a prerelease Python installed — without the flag the prerelease is skipped with a message and no wheel is built for it; with `--allow-prereleases` it's included again. Fall back to a crafted metadata fixture if no real prerelease interpreter is available.

---

## Testing Strategy

### Unit Tests

- [x] `test_interpreter_releaselevel_marks_prerelease` (in `discovery.rs`): feeds `from_metadata_message` a message with `releaselevel` of `alpha`, `beta`, and `candidate` and asserts each is flagged `is_prerelease`; `final` is asserted not prerelease. This covers the detection logic that everything else keys off.
- [x] Existing `discovery.rs`/`build_options.rs` suites still pass after threading the new field/flag through (updated the existing test fixtures to include `releaselevel`/`is_prerelease`).

### Integration Tests

- The filter itself (resolver dropping a prerelease unless `--allow-prereleases` is passed) is a small `.filter()` over the discovered list; I didn't add a full integration test for it because maturin's discovery probes real interpreters on the host and I don't have an alpha/beta Python installed to exercise the skip path deterministically. If a maintainer wants coverage there, the cleanest spot is a resolver-level test with an injected interpreter list.

### Manual Testing

Built the binary and verified end to end on my machine (Python 3.12.3, `final`):

- `python3 src/python_interpreter/get_interpreter_metadata.py` now includes `"releaselevel": "final"` (previously absent).
- `maturin build --help` and `maturin publish --help` now list `--allow-prereleases`.
- `cargo fmt --check`, `cargo clippy`, and the `python_interpreter`/`build_options` test suites all pass.

The one thing I couldn't exercise locally is the actual skip-a-prerelease behavior, since that needs a real prerelease interpreter on the host; the unit test above stands in for the detection half of it.

---

## Implementation Notes

### Week 1 Progress

I audited issue #1975 for claimability (checked assignee history via the Timeline API, searched for any competing PR, and ruled out a false-positive bot cross-reference from an unrelated repo), installed a Rust toolchain, and cloned the repo. Then I confirmed, through actually running the script rather than just reading the source, that `releaselevel` gets dropped at the very first stage of interpreter discovery (`get_interpreter_metadata.py`) and never reaches any Rust code. I posted a reproduction comment on the issue before starting Phase III, the same practice that worked well on my first two contributions of getting maintainer sign-off before writing code, and re-ran the same checks a week later against the latest code since I hadn't heard back yet.

### Week 2 Progress

Implemented the fix on branch `allow-prereleases`, following the data flow end to end: collect `releaselevel` in the probe script, carry it through `InterpreterMetadataMessage`, mark each discovered `PythonInterpreter` as prerelease-or-not, add the `--allow-prereleases` flag, and filter in the resolver. Added a unit test for the detection, updated existing test fixtures, and confirmed `fmt`/`clippy`/tests pass and the flag shows up in `--help`. Kept the diff deliberately small since the maintainers value easy-to-review changes. Still waiting on a maintainer reply to the issue; holding off on opening the PR until then / until I decide to submit.

### Code Changes

- **Files modified:**
  - `src/python_interpreter/get_interpreter_metadata.py` — collect `releaselevel`.
  - `src/python_interpreter/discovery.rs` — `releaselevel` field on `InterpreterMetadataMessage`; set `is_prerelease` in `from_metadata_message`; unit test.
  - `src/python_interpreter/mod.rs` — `is_prerelease` field on `PythonInterpreter`.
  - `src/python_interpreter/resolver.rs` — thread `allow_prereleases`; filter prereleases in `discover_native`.
  - `src/build_options.rs` — `--allow-prereleases` CLI flag on `PythonOptions`.
  - `src/build_context/builder.rs`, `src/develop/mod.rs` — pass the new flag through.
- **Key commits:** [Links to important commits — pending push]
- **Approach decisions:** field lives on `PythonInterpreter` not `InterpreterConfig` (runtime property, avoids a 15-site sprawling diff and bundled-config semantics); filter lives in the resolver not `find_all` (keeps `list-python` showing everything); explicit `-i` interpreters are never filtered (naming one is already opt-in).

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- Issue #1975: https://github.com/PyO3/maturin/issues/1975
- Issue timeline (assignment history): https://api.github.com/repos/PyO3/maturin/issues/1975/timeline
- Cross-referenced PR investigated and ruled out: https://github.com/CelestoAI/SmolVM/pull/146
- Repository: https://github.com/PyO3/maturin
- Relevant source: https://github.com/PyO3/maturin/blob/main/src/python_interpreter/discovery.rs
- Relevant source: https://github.com/PyO3/maturin/blob/main/src/python_interpreter/get_interpreter_metadata.py
- Cross-compile precedent for `releaselevel` parsing: https://github.com/PyO3/maturin/blob/main/src/cross_compile.rs
