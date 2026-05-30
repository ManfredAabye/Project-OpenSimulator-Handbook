# Prompt Collection: Review Source Code, Test It, and Resolve TODOs (OpenSimulator)

Base path for all prompts:

- D:/opensim-next/opensim

Usage notes:

- Replace placeholders like [FILE], [SECTION], [TODO_TEXT].
- These prompts are designed for Copilot/LLM workflows in VS Code.

## 1) Perform a functional review of a code section

```text
Analyze the code section in [FILE] at [SECTION].
Goal: identify functional risks, side effects, regressions, and missing edge-case handling.
Expected output in this order:
1. Critical findings (with severity, reason, exact location)
2. Medium findings
3. Low findings
4. Open assumptions/questions
5. Concrete change suggestions as short diff ideas
If there are no findings, explicitly write: No findings, and list remaining test gaps.
```

## 2) Review a code section for performance and scalability

```text
Review [FILE] section [SECTION] for performance issues under load.
Focus on:
- unnecessary allocations
- expensive loops/N+1 patterns
- locking/threading risks
- I/O in hot paths
Provide:
1. Prioritized bottlenecks
2. Expected impact (CPU, RAM, latency)
3. Concrete optimizations with a minimally invasive approach
4. Measurement plan (which metrics before/after the change)
```

## 3) Unit test plan for a code area

```text
Create a unit test plan for [FILE] section [SECTION].
Expected:
1. Test cases as a table: Name | Input | Expected | Why it matters
2. Required mocks/stubs
3. Edge cases and error paths
4. Minimal test data
5. Implementation order (quick wins first)
Use existing OpenSimulator project conventions.
```

## 4) Integration test plan for module boundaries

```text
Create an integration test plan for [MODULE_A] with [MODULE_B].
Context path: D:/opensim-next/opensim.
Provide:
1. Dependencies and interfaces
2. Test scenarios for success/failure/timeout/retry
3. Required configuration and test fixtures
4. Expected logs/signals for verification
5. Risks around concurrency and distributed state
```

## 5) Turn all TODOs in a file into concrete tasks

```text
Read all TODO comments in [FILE] and convert them into actionable tasks.
Format per TODO:
1. TODO text (original)
2. Problem in one sentence
3. Solution approach (technically concrete)
4. Acceptance criteria
5. Test strategy
6. Effort (S/M/L)
7. Risk (Low/Medium/High)
Sort by risk and impact.
```

## 6) Turn a TODO into a complete solution with patch

```text
Work on this TODO and propose a complete solution:
File: [FILE]
Line/Section: [SECTION]
TODO: [TODO_TEXT]

Provide in this order:
1. Short problem definition
2. Planned solution approach
3. Patch (as small as possible, no unnecessary refactoring)
4. Reasoning for why no regression is expected
5. Tests to add or update
```

## 7) Bulk TODO handling across multiple files

```text
I will provide a TODO list from multiple files in OpenSimulator.
Create a prioritized implementation roadmap.
Expected:
1. Clustering by topic (Security, Permissions, Networking, Performance, Cleanup)
2. Dependencies between TODOs
3. Recommended order in 2-week sprints
4. Risks and rollback plan per sprint
5. Definition of Done per cluster
```

## 8) Security-focused TODO review

```text
Review all TODOs in [FILE_OR_FOLDER] for security relevance.
Specifically identify missing checks for:
- permissions/authentication
- input validation
- privilege escalation
- information leakage in logs/error messages
Output:
1. Security TODOs (prioritized)
2. Concrete mitigations
3. Negative tests (abuse scenarios)
```

## 9) Prompt for generating test code

```text
Write tests for [FILE] and section [SECTION].
Requirements:
1. Follow the existing style of the test projects
2. Deterministic tests (no flaky sleeps)
3. Meaningful assert messages
4. Cover success path and failure path
5. Only required mocks
Provide a short test strategy first, then the complete test code.
```

## 10) Prompt for reviewing a proposed TODO fix

```text
Review the following fix for a TODO in OpenSimulator:
[PATCH_OR_CODE]

Check:
1. Correctness and completeness
2. Side effects in adjacent components
3. Thread safety and race conditions
4. Logging/error handling
5. Maintainability
Output findings by severity plus concrete follow-up improvements.
```

## 11) Prompt for a quick daily routine

```text
Run this routine for the OpenSimulator folder D:/opensim-next/opensim:
1. List the 10 most important open TODOs by risk x impact.
2. Pick the top 3 for today.
3. Provide a minimal solution sketch for each top TODO.
4. Generate the required tests for each top TODO.
5. Create a short completion checklist.
```

## 12) Prompt template with strict output format (machine-readable)

```text
Analyze TODOs in [FILE_OR_FOLDER] and respond only as JSON.
Schema:
{
  "items": [
    {
      "file": "...",
      "line": 0,
      "todo": "...",
      "category": "security|bug|perf|refactor|test|docs",
      "severity": "low|medium|high|critical",
      "proposal": "...",
      "acceptance_criteria": ["..."],
      "tests": ["..."],
      "effort": "S|M|L"
    }
  ]
}
No additional explanations outside JSON.
```
