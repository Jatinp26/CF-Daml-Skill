# Daml Skill for Claude

A Claude skill that gives any Claude/ Claude Code conversation correct knowledge of the Daml smart contract and Canton Network. Useful for developers writing, reviewing, or debugging Daml contracts.

## What it does

When you ask Claude about Daml templates, choices, authorization errors, multiparty workflows, testing with Daml Script, SDK setup, Canton 3.5 changes, the skill loads automatically and guides Claude to give correct, grounded answers rather than plausible-sounding guesses.

The skill is particularly good at catching the mistakes that aren't syntax errors but cause runtime authorization failures or incorrect ledger behavior, the things a compiler won't catch.

## Installation

1. Download `CF-daml-skill.skill`
2. In Claude.ai, go to **Settings → Skills**
3. Upload the `.skill` file
4. Start a new conversation, the skill is now active

## What's inside

```
CF-Daml-Skill/
├── SKILL.md                   
├── CF-daml-skill.skill                   
└── references/
    ├── language.md           
    ├── patterns.md            
    └── canton35.md           
```

**`SKILL.md`** loads on every relevant question. It covers:

- Core principles Claude must apply (authorization chain tracing, signatory vs observer vs controller, consuming vs non-consuming, etc.)
- A table of the most common Daml mistakes with correct alternatives
- `daml.yaml` quick reference for Canton 3.5
- `dpm` command cheat sheet
- A code review checklist for when someone pastes a file to review

**`references/language.md`** covers the full language: native types, records, variants, Optional, functions, control flow, templates, the authorization model (formal), choice anatomy, `createCmd` vs `create`, Daml Script testing API, interfaces, contract keys, and the standard library.

**`references/patterns.md`** covers 7 canonical Canton patterns with working code skeletons: Propose-Accept, Delegation, Authorization token, Locking, Multiple party agreement, Transfer proposal (3-party/issuer-controlled), and Observable event. Includes a pattern selection guide and notes on composing patterns.

**`references/canton35.md`** covers Canton 3.5 specifics: the `daml` > `dpm` CLI migration, Protocol Version 34 vs 35 differences, Daml-LF 2.3 and contract keys, breaking API changes (package-name addressing, scope based JWT deprecation), `daml.yaml` configuration, multi-package project structure, and common migration failures.

## What it's good at

**Catching authorization errors before they hit the ledger**

The most common Daml failure mode isn't a syntax error it's an authorization chain that doesn't add up. The skill traces who needs to authorize what, and whether the parent transaction covers it.

**Naming the right pattern**

Instead of improvising a workflow from scratch, the skill recognizes when Propose Accept, Delegation, or Multiple Party Agreement is the right structure, and gives you a correct skeleton rather than a plausible-looking one.

**`createCmd` vs `create` and other context-dependent syntax**

`createCmd` is only valid inside `submit` blocks in Daml Script. `create` is only valid inside choice `do` blocks. This trips up almost everyone new to Daml. The skill catches this and corrects it.

**Canton 3.5 / SDK 3.5 specifics**

The `daml` CLI is removed in SDK 3.5. Contract keys require `--target=2.3`. `daml-script` must not appear in production packages. Scope-based JWTs are deprecated. The skill knows all of this and applies it when relevant.

**Code review**

Paste a Daml file and ask for a review. The skill works through a checklist: module header, signatories, authorization chain, `ensure` clauses, choice return types, consuming/non-consuming correctness, `Cmd` suffix usage, test coverage.

## What it won't do

- It won't write your entire application for you without understanding the business logic
- It won't make authoritative claims about undocumented or rapidly-changing Canton Network APIs (it will say so)
- It won't replace running `dpm test` always test your code

## Triggers

The skill activates on questions about:

- Daml templates, choices, signatories, observers, controllers
- Authorization errors and why a transaction fails
- Daml Script tests (`submit`, `submitMustFail`, `queryContractId`, etc.)
- Multi-party workflows and which pattern to use
- `daml.yaml` configuration and project structure
- `dpm` commands (build, test, studio, sandbox, codegen)
- Contract keys, interfaces, data types
- Canton 3.5 / SDK 3.5 migration
- Translating Ethereum/Solidity patterns to Daml

## SDK version

Built against **Canton 3.5.3 / SDK 3.5.x**. Covers Daml-LF 2.2 (default) and 2.3 (with `--target=2.3`). References Protocol Version 34 and 35.

> ***Developed by Canton Foundation DevRel***
