# Contribution 3: Make `--find-interpreter` require opt-in to prereleases

**Contribution Number:** 3  
**Student:** Tien Nguyen (GitHub: [@tientnc](https://github.com/tientnc))  
**Issue:** [PyO3/maturin #1975](https://github.com/PyO3/maturin/issues/1975)  
**PR:** [PyO3/maturin #3276](https://github.com/PyO3/maturin/pull/3276)  
**Status:** Phase III - PR open, waiting on maintainer review

---

## Why I Chose This Issue

I picked `PyO3/maturin` issue #1975 to work outside Swift/Java, my first two contributions. maturin is a mature, active Rust project (5,600+ stars) that builds Python wheels for Rust code.

Before picking it, I checked it was actually open to work on: unassigned, labeled `good first issue`, no PR. There was some history: `fatelei` said "i want to have a try" in March 2024 and got assigned, but maintainer `konstin` unassigned them in January 2026, and no PR ever came from it. I also found a bot comment on an unrelated repo's PR (`CelestoAI/SmolVM#146`) mentioning the issue in passing - not real work on it, just a passing reference. Since `konstin` freed the issue back up, I treated it as available. (I double-checked `konstin` really is a maintainer here, not just someone with permission to unassign by accident - they have 100+ merged PRs on this repo.)

I posted a comment on the issue with my plan, but haven't heard back yet.

---

## Understanding the Issue

### Problem Description

`maturin build`/`publish` with `--find-interpreter` (`-f`) finds every installed Python and builds a wheel for it - final release or prerelease (alpha/beta/rc), no difference. Maintainer `davidhewitt` opened the issue, worried this could make people accidentally publish prerelease wheels to PyPI. (Also checked: he's reviewed 20+ merged PRs on this repo, so "maintainer" is accurate here too.)

### Expected Behavior

`--find-interpreter` should skip any interpreter that isn't a final release, unless the user passes a new `--allow-prereleases` flag. When it skips one, it should say so and mention the flag, not just silently drop it.

### Current Behavior

No prerelease filtering exists anywhere, and no `--allow-prereleases` flag exists.

### Affected Components

- `get_interpreter_metadata.py`: the script maturin runs inside each Python to collect info. Never reads `sys.version_info.releaselevel`.
- `discovery.rs`: `InterpreterMetadataMessage` has no `releaselevel` field, so `from_metadata_message()` has nothing to check.
- `build_options.rs`: no `allow_prereleases` field or flag.
- `cross_compile.rs`: `releaselevel` shows up here, but only in test fixtures. The actual parser (`BuildDetailsImplementation`) only reads `name`, so the field isn't used anywhere in the codebase yet.

---

## Reproduction Process

### Environment Setup

No Rust installed, so I installed it with `rustup` (no `sudo` needed), then cloned maturin and built it with `cargo build`. No fork yet - this was investigation, not code.

### Steps to Reproduce

1. `python3 -c "import sys; print(sys.version_info)"` - Python already reports `releaselevel` (e.g. `final`).
2. `python3 src/python_interpreter/get_interpreter_metadata.py` - its output has no `releaselevel` key.
3. `grep -rn "releaselevel" src/` - only shows up in one unrelated test file, never in the actual discovery code.
4. `maturin build --help | grep -i prerelease` - the flag doesn't exist.

**Result:** the feature just hasn't been built yet - not a partial fix, not something that broke later.

Repeated all four steps again on 2026-07-14 after pulling 8 new commits, to make sure nothing had changed. Same result.

### Reproduction Evidence

- **Commit showing reproduction:** none - reproduced locally via the commands and logs below, not as a committed script.
- **Logs (checked on commit `e54b7775`, then again on `8f981619`, same result both times):**
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
- **My findings:** the info is dropped at the very first step - the small probe script - so nothing downstream can use it even if it wanted to. That's where the fix has to start.

---

## Solution Approach

### Analysis

Every Python already knows if it's a final release or a preview (alpha/beta/rc) - that's what `sys.version_info.releaselevel` is for. maturin just never asks for it. The probe script collects basic info like the version number but skips `releaselevel`, so nothing downstream ever gets that data, even if it wanted to filter on it.

More precisely: `get_interpreter_metadata.py` never reads `releaselevel`, so `InterpreterMetadataMessage` has no field for it, and `from_metadata_message()`/`find_all()` have nothing to check. The fix has to start at that first collection point.

### Proposed Solution

Fix it in the order the information flows:

1. **Collect it.** Add `releaselevel` to the probe script.
2. **Carry it.** Add the field to `InterpreterMetadataMessage`, and mark each discovered interpreter as prerelease or not.
3. **Filter it.** In the resolver - the one place that knows both which interpreters were found and which flags the user passed - drop prereleases and print a notice, unless `--allow-prereleases` is set. Interpreters picked explicitly with `-i` are never filtered, since naming one by hand is already an opt-in.
4. **Add the flag.** `--allow-prereleases`.

Filtering in the resolver (not lower down) keeps the skip logic and its warning message in one place.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** `-f/--find-interpreter` builds for every Python found, prereleases included. Need to detect prereleases and make them opt-in via a new flag, without changing behavior for interpreters named explicitly with `-i`.

**Match:** `gil_disabled` is basically the same pattern already in the codebase - a bool that starts in the probe script, flows through `InterpreterMetadataMessage`, and gets used on the Rust side. I followed that same path for `releaselevel`. `from_metadata_message()` also already skips-and-warns for other reasons (old Python versions, wrong Windows architecture), so I copied that style. (`releaselevel` also shows up in `cross_compile.rs` test fixtures, but that parser doesn't read it, so it's unrelated.)

**Plan:**
1. `get_interpreter_metadata.py`: add `releaselevel` to the output.
2. `discovery.rs`: add `releaselevel` to `InterpreterMetadataMessage`; compute `is_prerelease` in `from_metadata_message()`.
3. `mod.rs`: add `is_prerelease` to `PythonInterpreter` - not to `InterpreterConfig`, since that struct is built in ~15 places from bundled sysconfig data, so adding it there would be a much bigger, messier change.
4. `build_options.rs`: add the `--allow-prereleases` flag.
5. `resolver.rs`: pass the flag through; filter prereleases in `discover_native()` unless it's set, printing a notice for each one skipped.
6. Tests: unit test for the alpha/beta/candidate/final detection logic.

**Implement:** Done on branch `allow-prereleases`. Small diff: 85 insertions / 3 deletions across 7 files (most of it one line - `is_prerelease: false` - repeated across existing constructors Rust forces you to update). Two decisions worth flagging: `is_prerelease` lives on `PythonInterpreter`, not `InterpreterConfig`, for the reason above; and the filter lives in the resolver, not `find_all()`, so `maturin list-python` (which also calls `find_all()`) still shows every interpreter.

**Review:** Ran `cargo fmt`, `cargo clippy`, and the test suite before opening the PR. Kept the diff small on purpose - checked recent merged PRs and the maintainers clearly favor small, easy-to-review changes.

**Evaluate:** Built maturin locally and confirmed the probe script now reports `releaselevel`, and `--allow-prereleases` shows up in `--help`. I couldn't test the actual skip-a-prerelease behavior end to end - no prerelease Python installed locally - so the unit test covers detection only, not the full skip path.

---

## Testing Strategy

### Unit Tests

- [x] `test_interpreter_releaselevel_marks_prerelease` (`discovery.rs`): checks `alpha`/`beta`/`candidate` are flagged prerelease, and `final` is not.
- [x] Existing `discovery.rs`/`build_options.rs` tests still pass after adding the field/flag (updated their fixtures).

### Integration Tests

- Didn't add one for the filter itself (resolver dropping a prerelease unless `--allow-prereleases` is passed) - that needs a real prerelease Python on the host, and I don't have one. A resolver test with a fake/injected interpreter list would be the way to cover it.

### Manual Testing

Verified on my machine (Python 3.12.3, `final`):
- probe script output now includes `"releaselevel": "final"`
- `--allow-prereleases` shows up in `maturin build --help` and `maturin publish --help`
- `cargo fmt --check`, `cargo clippy`, and the relevant test suites all pass

Couldn't test the actual skip behavior locally - no prerelease Python installed.

---

## Implementation Notes

### Week 1 Progress

Checked issue #1975 was actually claimable (assignee history, no competing PR, ruled out an unrelated bot comment), installed Rust, cloned the repo. Confirmed by actually running the code - not just reading it - that `releaselevel` gets dropped at the first step and never reaches Rust. Posted a reproduction comment before writing code, same as my first two contributions, and re-checked a week later since I hadn't heard back.

### Week 2 Progress

Implemented the fix on branch `allow-prereleases`: collect `releaselevel` in the probe script, carry it through `InterpreterMetadataMessage`, mark interpreters as prerelease or not, add `--allow-prereleases`, filter in the resolver. Added a unit test, updated existing fixtures, confirmed fmt/clippy/tests pass. Kept the diff small on purpose. Forked the repo, pushed the branch, and opened PR #3276. Still no reply on the original issue.

### Code Changes

- **Files modified:**
  - `get_interpreter_metadata.py` - collect `releaselevel`
  - `discovery.rs` - `releaselevel` on the message struct; set `is_prerelease`; unit test
  - `mod.rs` - `is_prerelease` field on `PythonInterpreter`
  - `resolver.rs` - thread `allow_prereleases`; filter in `discover_native`
  - `build_options.rs` - `--allow-prereleases` flag
  - `build_context/builder.rs`, `develop/mod.rs` - pass the flag through (Rust requires these call sites to update)
- **Key commits:** see PR [#3276](https://github.com/PyO3/maturin/pull/3276)
- **Approach decisions:** `is_prerelease` on `PythonInterpreter`, not `InterpreterConfig` (avoids touching ~15 constructors, and bundled configs are always released versions anyway); filter in the resolver, not `find_all()` (keeps `list-python` unfiltered); explicit `-i` interpreters are never filtered.

---

## Pull Request

**PR Link:** https://github.com/PyO3/maturin/pull/3276

**PR Description:** adapted from the Solution Approach section above, following maturin's PR template.

**Maintainer Feedback:**
- None yet.

**Status:** Awaiting review.

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
- PR #3276: https://github.com/PyO3/maturin/pull/3276
- Issue timeline (assignment history): https://api.github.com/repos/PyO3/maturin/issues/1975/timeline
- Cross-referenced PR investigated and ruled out: https://github.com/CelestoAI/SmolVM/pull/146
- Repository: https://github.com/PyO3/maturin
- Relevant source: https://github.com/PyO3/maturin/blob/main/src/python_interpreter/discovery.rs
- Relevant source: https://github.com/PyO3/maturin/blob/main/src/python_interpreter/get_interpreter_metadata.py
- Cross-compile file checked and ruled out: https://github.com/PyO3/maturin/blob/main/src/cross_compile.rs
