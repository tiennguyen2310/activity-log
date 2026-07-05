# Contribution 3: Make `--find-interpreter` require opt-in to prereleases

**Contribution Number:** 3  
**Student:** Tien Nguyen (GitHub: [@tientnc](https://github.com/tientnc))  
**Issue:** [PyO3/maturin #1975](https://github.com/PyO3/maturin/issues/1975) (Fork: not yet created)  
**Status:** Phase II Complete (reproduction confirmed; implementation not yet started)

---

## Why I Chose This Issue

I chose `PyO3/maturin` issue #1975 to get experience outside the Swift/Java ecosystem from my first two contributions. `maturin` is a mature Rust project (5,600+ stars) that builds Python wheels for Rust extensions, and picking it let me learn Rust CLI conventions along with Python's `sysconfig`/`sys.version_info` internals.

Before selecting it, I checked whether it was actually claimable: open, labeled `good first issue`, and unassigned. There was some history to work through, though. `fatelei` commented "i want to have a try" back in March 2024 and got assigned by maintainer `messense`, but maintainer `konstin` unassigned `fatelei` on 2026-01-05 (I confirmed this through the GitHub Timeline API, which turned out to be more reliable here than the `/events` REST endpoint), and no PR ever came out of it (`search/issues?q=repo:PyO3/maturin+1975+type:pr` returns 0 results). I also chased down a cross-reference that looked like competing work at first: a bot review comment on an unrelated repo's PR (`CelestoAI/SmolVM#146`) mentioned the issue in passing, but it wasn't actual work on the fix. Since a maintainer had deliberately freed the issue up, I treated it as genuinely available.

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
- One interesting wrinkle: `src/cross_compile.rs` already parses `releaselevel` from a target's `build-details.json` for *cross-compilation* targets, but only on that separate path, and the value gets discarded instead of enforced. So the concept already exists in the codebase, it's just not wired into native discovery or enforcement anywhere.

---

## Reproduction Process

### Environment Setup

No Rust toolchain was on my machine, so I installed one with `rustup` (`curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -y --default-toolchain stable --profile default`), which resolved to Rust 1.96.1. No `sudo` was needed, the same constraint I had during my Swift-Java setup. I cloned `https://github.com/PyO3/maturin.git` directly with no fork yet, since I haven't started implementation. `cargo build` finished cleanly in about 3 minutes at HEAD commit `e54b7775`, with no missing system dependencies.

### Steps to Reproduce

1. Confirm the real interpreter reports prerelease info natively: `python3 -c "import sys; print(sys.version_info)"` → `sys.version_info(major=3, minor=12, micro=3, releaselevel='final', serial=0)`.
2. Run maturin's own probe script against that same interpreter: `python3 src/python_interpreter/get_interpreter_metadata.py` → JSON output with `major`/`minor`/`abiflags`/etc., but **no `releaselevel` key at all**.
3. Search the codebase for any existing prerelease handling in the native-discovery path: `grep -rn "releaselevel\|allow_prerelease" src/`. The only hits are in `cross_compile.rs`'s cross-compilation-target test fixtures, nothing in `discovery.rs` or `build_options.rs`.
4. Check the built CLI directly: `./target/debug/maturin build --help | grep -i prerelease` → no output; the flag does not exist.
5. **Observed:** the feature described in the issue is completely unimplemented on current `main`. It's not a regression and not partially done, it's just missing.

### Reproduction Evidence

- **Commit showing reproduction:** none yet. I haven't forked or branched since this is Phase II (investigation), not Phase III (implementation).
- **Logs:**
  ```
  $ python3 -c "import sys; print(sys.version_info)"
  sys.version_info(major=3, minor=12, micro=3, releaselevel='final', serial=0)

  $ python3 src/python_interpreter/get_interpreter_metadata.py
  {"implementation_name": "cpython", "executable": "...", "major": 3, "minor": 12,
   "abiflags": "", "interpreter": "cpython", "ext_suffix": "...", "soabi": "...",
   "platform": "linux-x86_64", "system": "linux", "pointer_width": 64, "gil_disabled": false}

  $ ./target/debug/maturin build --help | grep -i -A2 "find-interpreter\|prerelease"
    -f, --find-interpreter
            Find interpreters from the host machine
  ```
- **My findings:** the gap isn't a missed filter condition somewhere in the Rust code. The prerelease information is dropped at the very first step, the Python-side probe script, so nothing downstream in Rust could distinguish a prerelease interpreter even if it wanted to. That tells me the actual first step of the fix: add the field at the source before any Rust-side logic can use it.

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week 1 Progress

I audited issue #1975 for claimability (checked assignee history via the Timeline API, searched for any competing PR, and ruled out a false-positive bot cross-reference from an unrelated repo), installed a Rust toolchain, and cloned the repo. Then I confirmed, through actually running the script rather than just reading the source, that `releaselevel` gets dropped at the very first stage of interpreter discovery (`get_interpreter_metadata.py`) and never reaches any Rust code. I'm posting a reproduction comment on the issue before starting Phase III, the same practice that worked well on my first two contributions of getting maintainer sign-off before writing code.

### Week [X] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

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
