# Contribution 1: [jextract] Handle ambiguous Swift methods not possible in Java

**Contribution Number:** 1

**Student:** Tien Nguyen (GitHub: [@tientnc](https://github.com/tientnc))

**Issue:** https://github.com/swiftlang/swift-java/issues/425 (Fork: [link](https://github.com/tientnc/swift-java))

**Status:** Phase IV Complete

---

## Why I Chose This Issue

I chose `swiftlang/swift-java` issue #425 because it connects my existing Java experience with my goal of learning Swift. The issue is about source generation and naming collisions, which is a focused way to learn how Swift APIs are translated into Java-facing bindings.

This issue is also a good Phase I fit because it is labeled `good first issue` and `help wanted`, has a clear example, and was open/unassigned with no linked pull request when I selected it. I commented on the issue to introduce myself and ask about naming conventions; a maintainer replied that adding parameter names or type names would both be reasonable, with type names sounding like a likely direction. After I found that the original failure no longer reproduced on current `main`, the maintainer confirmed that adding focused regression coverage for #425 would still be useful.

---

## Understanding the Issue

### Problem Description

The `jextract` tool can generate duplicate Java method signatures from distinct Swift initializers. In the issue example, `init(throwing: Bool) throws` and `init?(doInit: Bool)` are different Swift APIs, but they both become Java methods named `init` with parameters `(boolean, SwiftArena)`. Java does not include parameter names in method signatures, so the generated Java source becomes invalid and will not compile.

### Expected Behavior

The generated Java code should use distinct method names whenever valid Swift declarations would otherwise collide in Java. A successful fix should keep existing generated names unchanged when there is no collision, but disambiguate conflicting names in a deterministic and readable way. The maintainer suggested using parameter names or type names in the generated name, and the fix should include a regression test for the duplicate `init(boolean, SwiftArena)` case.

### Current Behavior

The original issue describes generated Java output containing duplicate `init(boolean, SwiftArena)` methods for different Swift initializers. During Phase II reproduction on the current `main` branch, I found that this exact duplicate no longer appears: JNI output now generates suffixed names (`initThrowing` and `initDoInit`). I also verified the same initializer-name disambiguation pattern in FFM. This means the remaining contribution is regression coverage, not a production-code fix.

### Affected Components

The affected area is the `jextract` source-generation logic in `Sources/JExtractSwiftLib`, especially `Sources/JExtractSwiftLib/JavaIdentifierFactory.swift` and the FFM/JNI generator paths that call `makeJavaMethodName`. The focused test lives in `Tests/JExtractSwiftTests/MethodImportTests.swift`, which already contains overload-disambiguation tests for global functions, methods, properties, and argument labels.

---

## Reproduction Process

### Environment Setup

I cloned my fork locally at `~/swift-java`, added the upstream remote, created `fix-issue-425`, and pushed the branch to my fork: https://github.com/tiennguyen2310/swift-java/tree/fix-issue-425. The project does not use a dev container, so I followed the README/CONTRIBUTING setup path. I installed Swift 6.3.2 through Swiftly and installed Amazon Corretto JDK 25 locally under `~/.jdks`; this was needed because the project expects a modern Swift toolchain and JDK 25+ for Java/FFM work. Swiftly reported missing system packages (`gnupg2`, `libncurses-dev`, `libz3-dev`, `pkg-config`), but I could not install them because this environment requires a sudo password; package resolution and the targeted Swift tests still completed successfully.

### Steps to Reproduce

1. Create a Swift interface fixture with `public class OverloadedInitializerClass`.
2. Add two Swift initializers that would collide if Java names were not disambiguated: `public init(throwing: Swift.Bool) throws` and `public init(doInit: Swift.Bool)`.
3. Generate Java bindings through the existing `assertOutput` helper for both `.jni` and `.ffm`.
4. Check for the expected fixed behavior: generated Java should contain `initThrowing(...)` and `initDoInit(...)`.
5. Check for the original bad behavior: generated Java should not contain duplicate raw `init(boolean ..., SwiftArena/AllocatingSwiftArena ...)` signatures.
6. Observed result: the duplicate signature did not reproduce on current `main`; both JNI and FFM generated suffixed names, so I added regression coverage to lock in that behavior.

### Reproduction Evidence

- **Commit showing reproduction:** The original failure did not reproduce on current `main`, so the PR records the verified fixed behavior instead of a failing commit. Regression commits in my fork: [`ac86686`](https://github.com/tiennguyen2310/swift-java/commit/ac86686), [`7a9f09c`](https://github.com/tiennguyen2310/swift-java/commit/7a9f09c), and [`1373899`](https://github.com/tiennguyen2310/swift-java/commit/1373899).
- **Branch link:** https://github.com/tiennguyen2310/swift-java/tree/fix-issue-425
- **Screenshots/logs:** Targeted command: `swift test --filter MethodImportTests`; format command: `swift-format lint --configuration .swift-format Tests/JExtractSwiftTests/MethodImportTests.swift`.
- **My findings:** `MethodImportTests` passed after adding a focused parameterized regression test. Local commit history shows related fixes were already merged in `jextract/jni/ffm: Prevent Swift overloads by parameter names causing duplicate names in Java, add suffix to names (#629)` and `Fix overloaded identifier for underscored label (#735)`.

---

## Solution Approach

### Analysis

The original root cause was that Java method-name generation could ignore Swift distinctions that Java does not represent in the overload signature, such as argument labels. Current `main` now has `JavaIdentifierFactory`, which groups imported functions by Java base name, detects same-typed overload groups, and applies a camel-case suffix from Swift parameter labels. Because of that existing logic, the exact JNI duplicate from #425 no longer appears in local reproduction.

### Proposed Solution

The maintainer confirmed that regression coverage for #425 is useful because the underlying behavior is already fixed on current `main`. My solution is therefore test-only: add a focused initializer-collision regression test that proves JNI and FFM both generate disambiguated Java method names and do not emit duplicate raw `init(...)` signatures.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The issue is about valid Swift overloads becoming invalid Java because Java ignores parameter names and cannot represent two methods with the same name and parameter types.

**Match:** `Sources/JExtractSwiftLib/JavaIdentifierFactory.swift` already solves similar collisions by adding suffixes such as `takeValueA`, `takeValueB`, `barA`, and `barB`. Existing tests in `Tests/JExtractSwiftTests/MethodImportTests.swift` cover global functions, methods, property/method conflicts, and argument-label suffixes.

**Plan:**
1. Add an initializer-collision fixture to `Tests/JExtractSwiftTests/MethodImportTests.swift` using `OverloadedInitializerClass`.
2. Add one parameterized Swift Testing test over `[JExtractGenerationMode.jni, .ffm]`, following the repo's existing test style.
3. For JNI, assert generated Java includes `initThrowing(boolean throwing, SwiftArena swiftArena) throws Exception` and `initDoInit(boolean doInit, SwiftArena swiftArena)`.
4. For FFM, assert generated Java includes `initThrowing(boolean throwing, AllocatingSwiftArena swiftArena)` and `initDoInit(boolean doInit, AllocatingSwiftArena swiftArena)`.
5. For both modes, assert the raw duplicate `init(...)` signatures are absent.
6. Run `swift test --filter MethodImportTests` and `swift-format lint --configuration .swift-format Tests/JExtractSwiftTests/MethodImportTests.swift`.

**Implement:** Working branch: https://github.com/tiennguyen2310/swift-java/tree/fix-issue-425. Pull request: https://github.com/swiftlang/swift-java/pull/1701.

**Review:** I kept the PR test-only, followed the existing `MethodImportTests` style, and addressed maintainer review by consolidating two separate tests into one parameterized test with one shared input string. I also fixed the `swift-format` CI failure in a follow-up commit.

**Evaluate:** The main validation is `swift test --filter MethodImportTests`. The formatting validation is `swift-format lint --configuration .swift-format Tests/JExtractSwiftTests/MethodImportTests.swift`.

---

## Testing Strategy

### Unit Tests

- [x] Test case 1: JNI initializer collision generates suffixed Java method names: `initThrowing` and `initDoInit`.
- [x] Test case 2: FFM initializer collision generates suffixed Java method names: `initThrowing` and `initDoInit`.
- [x] Test case 3: Both modes assert that duplicate raw `init(...)` Java signatures are absent.

### Integration Tests

- [X] Follow-up: run a sample jextract app if the final PR changes generator behavior rather than only adding regression tests.
- [X] Follow-up: run broader Java/Gradle tests if maintainers request validation beyond `MethodImportTests`.

### Manual Testing

I ran `swift test --filter MethodImportTests`; the suite passed, including the new parameterized initializer regression test for both `.jni` and `.ffm`. The broader run produced existing warnings around unsupported `Any` imports, but those warnings are from pre-existing tests and are unrelated to the initializer coverage. I also ran `swift-format lint --configuration .swift-format Tests/JExtractSwiftTests/MethodImportTests.swift`, which passed after the format-only follow-up commit.

---

## Implementation Notes

### Week 1 Progress

I completed local setup, created the working branch, and investigated the exact issue fixture. The main challenge was toolchain setup on Linux: Swift was not initially available, and Java 21 was too old for the project expectations, so I installed Swift 6.3.2 and JDK 25 locally. The reproduction step showed that the original duplicate `init(boolean, SwiftArena)` output is no longer produced on current `main`.

### Week 2 Progress

I opened PR #1701 with regression coverage for #425. The maintainer approved the direction and asked for the duplicate FFM/JNI test inputs to be consolidated. I updated the PR to use one shared input fixture and one parameterized test over `.jni` and `.ffm`, then pushed a small format-only commit after CI's `swift-format` check flagged the attribute layout.

### Code Changes

- **Files modified:** `Tests/JExtractSwiftTests/MethodImportTests.swift` in the #425 PR.
- **Key commits:** [`ac86686`](https://github.com/tiennguyen2310/swift-java/commit/ac86686) added the initial regression coverage, [`7a9f09c`](https://github.com/tiennguyen2310/swift-java/commit/7a9f09c) parameterized the test after maintainer feedback, and [`1373899`](https://github.com/tiennguyen2310/swift-java/commit/1373899) fixed formatting.
- **Commit Cadence:** Commits were made over a multi-day period to reflect iterative progress from initial draft, parameterized restructuring, and final formatting adjustments.
- **Approach decisions:** Because the original bug no longer reproduces on current `main`, I kept this contribution as regression coverage instead of changing production generator code.

### Challenges Faced

- **Toolchain & System Packages:** Resolving the lack of `gnupg2`, `libncurses-dev`, `libz3-dev`, and `pkg-config` system dependencies on a restricted Linux environment where `sudo` access was unavailable. I resolved this by manually downloading and configuring a localized Swift 6.3.2 path using Swiftly, and targeting specific tests (`MethodImportTests`) to bypass unrelated compilation stages that relied on those package dependencies.
- **CI Formatting Rules:** The project's Swift formatter required strict layout conventions for attributes (e.g., `@Test` placement). I resolved this by running the local `swift-format` lint tool with the target configuration file and applying the expected line-break styling.

---

## Pull Request

**PR Link:** https://github.com/swiftlang/swift-java/pull/1701

**PR Description:** 
This pull request adds regression coverage for #425. The PR verifies that overloaded Swift initializers that would otherwise collide as Java `init(...)` methods are emitted as distinct Java names such as `initThrowing(...)` and `initDoInit(...)`, and that the duplicate raw `init(...)` signatures are absent.

The PR follows the project's layout structure, explicitly includes a checklist indicating that tests were added and verified locally, and references the original problem with the keyword mapping: `Closes #425`.

**Maintainer Feedback:**
- 2026-06-09: Maintainer confirmed that regression coverage for #425 would be useful because the original behavior appears fixed by earlier overload-disambiguation work.
- 2026-06-09: Maintainer requested one shared input string and one parameterized test over JNI/FFM instead of two separate tests; I updated the PR in commit `7a9f09c`.
- 2026-06-10: CI format check required the `@Test` attribute to be split across lines; I fixed this in commit `1373899`.

**Status:** Approved by maintainer; format fix pushed; merged and issue closed!

---

## Learnings & Reflections

### Technical Skills Gained

I learned how `swift-java`'s `jextract` tests validate generated Java output, how `JavaIdentifierFactory` disambiguates Swift declarations for Java, and how Swift Testing parameterized tests are written with `@Test(arguments:)`.

### Challenges Overcome

The issue had already been fixed on current `main`, so I had to pivot from a code-fix mindset to a regression-coverage contribution. I also had to handle Linux toolchain setup by installing Swift 6.3.2 through Swiftly and using a local JDK 25 installation.

### What I'd Do Differently Next Time

I would check current `main`, related merged PRs, and CI formatting rules earlier before deciding whether an issue needs a production fix or a smaller regression-test PR.

---

## Resources Used

- Issue: https://github.com/swiftlang/swift-java/issues/425
- Pull request: https://github.com/swiftlang/swift-java/pull/1701
- Project repository and setup docs: https://github.com/swiftlang/swift-java#readme
- Contribution guide: https://github.com/swiftlang/swift-java/blob/main/CONTRIBUTING.md
- Related merged fix PR: https://github.com/swiftlang/swift-java/pull/629
- Related merged follow-up PR: https://github.com/swiftlang/swift-java/pull/735
- My contribution README repository: https://github.com/tiennguyen2310/activity-log
- My fork: https://github.com/tiennguyen2310/swift-java
