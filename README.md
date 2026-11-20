# Harness Engineering with Claude & Claude Code

A capstone project demonstrating advanced AI system architecture across four integrated systems: **agentic loops**, **context optimization**, **multi-surface configuration**, and **orchestration**.

## Systems Overview

| System | Focus | Tests | Approx Cost |
|--------|-------|-------|------|
| **1. Agentic Loop** | Stop-reason-driven control flow for claim intake | 30  | $0.1251 |
| **2. Context Strategy** | 56% token reduction while preserving answer quality | 30  | $0.2127 |
| **3. Claude Code Config** | Hierarchy, path-scoped rules, and forked skills | 35 | $0.0000 |
| **4. Orchestration** | Tiered state, crash recovery, session forking | 33 | $0.0000 |

**Total API Spend:** $0.3378 USD | **Total Tests:** 128 passing

## Key Takeaways

### Loop Control (System 1)
- Termination driven by `stop_reason` field (tool_use → loop continues, end_turn → returns)
- Budget enforcement prevents infinite loops; anti-patterns tested explicitly
- Structured tool errors enable intelligent retry routing

### Context Optimization (System 2)
- **Active segments** preserved verbatim (15,789 tokens); **resolved segments** compressed (797 tokens compressed from 23,809 raw)
- Compression cached from prior runs; no inference cost per evaluation
- Control-variant tests prove case-facts block is load-bearing for structured questions

### Configuration as Code (System 3)
- Path-scoped rules with glob patterns eliminate cross-directory duplication
- Skills run in forked context with read-only allowed-tools to contain blast radius
- Validator ensures all imports resolve and no invalid hierarchy exists

### Orchestration (System 4)
- SQL-filtered warm-tier queries (last N hours) replace full-history loads
- Crash recovery: resume within 30-min staleness window, else fresh-start with summary
- Hot-state stays under 5 KB; resolved findings move to warm (SQL) and cold (archive) tiers

## Project Structure

```
├── Build a Claims Intake Agent with a stop_reason-Driven Loop/
│   └── exercises/03-dynamic-decomposition/solution/
├── Engineer a Long-Conversation Context Strategy for a Retail Support Copilot/
│   └── solution/ (budget.json, eval results)
├── Configure Claude Code for a Multi-Surface Monorepo Team/
│   └── solution/ (CLAUDE.md, path-scoped rules)
├── Build a Multi-Shift Quality Monitoring System with Claude Orchestration/
│   └── 04-fork-scratchpad/solution/
└── evidence/ (run artifacts, test logs, JSON outputs)
```

## Running the Systems

Each system includes a Python 3.13 environment and test suite. Example:

```bash
cd "Build a Claims Intake Agent with a stop_reason-Driven Loop"
pytest exercises/03-dynamic-decomposition/solution/ -v
```

## Rubric Compliance

**Automated testing:** 30+30+35+33 = 128 tests passing  
**Evidence artifacts:** Run logs, JSON traces, budget files, config validation  
**Reflection brief:** `my-reflection.md` (20 questions, 95 lines, all evidence-cited)  
**API spend:** Itemized per-system breakdown ($0.3378 total)

---

**Author:** Mahak-Accenture | **Date:** November 2026 | **Status:** Submission-ready
