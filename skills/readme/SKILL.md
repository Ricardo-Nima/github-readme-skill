---
name: github-readme
description: Generate high-quality README files for GitHub repositories. Supports two modes: interview mode (default, full 6-phase pipeline) and direct mode (--mode direct, skip interview and auto-generate). Triggers: /readme, generate README, write README, update README, github-readme.
argument-hint: "[--mode interview|direct] [--lang zh,en,ja,...] [--force] [project directory]"
level: 3
---

# GitHub README Generation Skill

<Purpose>
Generate a well-structured, code-consistent, and highly readable README.md file for any GitHub repository. Using a dual-engine approach driven by code analysis (ground truth) and user interview (user intent), the skill ensures the README accurately reflects the codebase while conveying the project's proper narrative. Supports multilingual translation and GitHub extended syntax.
</Purpose>

<Use_When>
- User says "/readme" or "generate README" or "write README" or "update README" or "generate README directly"
- User wants a high-quality GitHub README for their project
- User needs to restructure an existing README (preserve user narrative + update technical facts)
- User needs multilingual README versions
- New project initialization, need to create README from scratch
</Use_When>

<Do_Not_Use_When>
- User only needs to modify a specific paragraph in the README (use Edit tool directly)
- User only needs to add a badge or link (use Edit tool directly)
- User needs documentation on non-GitHub platforms (refer to other documentation skills)
- User explicitly says "don't use skill" or "write README directly"
</Do_Not_Use_When>

<Execution_Policy>
- Execute the 6 Phases strictly in order, do not skip any Phase
- Write checkpoint file (.readme-checkpoint-{N}.md) after each Phase, supporting resume on interruption
- Code analysis results take priority over user assumptions: when conflicts are found, code is authoritative and the user is notified
- `--mode direct` skips Phase 2 interview, auto-infers project narrative from code; `--mode interview` (default) keeps the full interview pipeline
- All configuration values must be replaced with placeholders (e.g., `<YOUR_API_KEY>`), never expose real keys
- Never write the final README file until Phase 4 user approval
- Interview asks only one question at a time, never batch questions
- Context budget strictly follows the file read limits per project size
- If Phase 1 code analysis fails (e.g., cannot identify project structure), degrade to interview-only mode and note it
</Execution_Policy>

<Execution_Modes>
This skill supports two execution modes, specified via the `--mode` parameter:

### interview (default)
Full 6-phase pipeline: Security Gate → Code Analysis → Bidirectional Interview → Generation + Cross-Validation → User Review → Multilingual Translation.
- User provides project narrative through single-question interviews (positioning, audience, installation, usage, language)
- Gap Check mechanism verifies consistency between user claims and code facts
- Suitable for projects requiring precise positioning, complex narratives, or first-time README generation
- Default mode, used when `--mode` is not specified

### direct
Skip Phase 2 interview, auto-infer project narrative from code.
- **Project description**: Auto-extracted from `package.json` description / `pyproject.toml` description / entry file comments / directory name
- **Target audience**: Defaults to "Developers"
- **Installation commands**: Auto-generated based on detected package manager (`npm install` / `pip install` / `cargo build` etc.), tagged `[Auto-generated]`
- **Usage examples**: Minimum viable examples generated from API signatures extracted in Phase 1, tagged `[Auto-generated]`
- **Multilingual**: Only generates source language version (no translation)
- Suitable for quick initial README drafts, simple projects, or when user wants to see results first and manually edit later
- Usage: `/readme --mode direct` or `generate README --mode direct`
</Execution_Modes>

<Secrets_Exclusion>
The following file types must NEVER be read by the Read tool:
- `.env`, `.env.*`, `.env.local`, `.env.development`, `.env.production`
- `credentials.*`, `credentials.json`, `service-account*.json`
- `*.pem`, `*.key`, `*.p12`, `*.pfx`
- `secrets.*`, `secrets.yaml`, `secrets.yml`
- `id_rsa*`, `id_ecdsa*`, `id_ed25519*`
- `*.token`, `private.*`, `*.private`
- Any file with `secret`, `credential`, `private_key` in its path
- `.npmrc` (if it contains `_authToken`)
</Secrets_Exclusion>

<Context_Budget>
Project size is determined by counting source files via Glob. Source file definition: `*.{py,js,ts,jsx,tsx,go,rs,java,rb,php,c,cpp,h,swift,kt,cs,scala,r,rs,ex,exs}`, excluding `node_modules`, `.git`, `dist`, `build`, `vendor`, `target`, `.next`, `__pycache__`, `.venv`, `venv`.

| Size | Criterion | Phase 1 File Read Limit | Strategy |
|------|-----------|------------------------|----------|
| Small | Source files <= 50 | 15 files | Full analysis: entry files + all config + directory tree + key source files |
| Medium | Source files 50-200 | 10 files | Sampled analysis: package manifest + entry files + top-level config + key directories |
| Large | Source files > 200 | 6 files | Summary analysis: only package manifest + top-level config + entry file sampling |
| Empty | Source files = 0 | 0 | Degrade to interview-only mode, note "No code detected" |
</Context_Budget>

<GitHub_Extended_Syntax>
Generated READMEs should use the following GitHub extended syntax (intelligently selected based on project characteristics):

**Required (all READMEs)**:
1. **Badges** (`<div align="center"><img alt="...">...</div>`): Tests, License, Version, Language. MUST wrap all badges in `<div align="center">` to render them horizontally centered. Badges are placed inline (no blank lines between `<img>` tags) inside the centered div.
2. **Code blocks** (with language annotation: ```python, ```bash, etc.)
3. **Tables**: API reference, configuration options, dependency lists
4. **Anchor links**: Table of contents navigation
5. **Language switcher** (MUST include if generating >= 2 languages): A `<p align="center">` block between the title/badges and the TOC, linking to all language versions with flag emojis.

**Conditional**:
5. **Alerts** (`> [!NOTE]`, `> [!WARNING]`, `> [!IMPORTANT]`, `> [!CAUTION]`): Select appropriate type based on content
6. **Collapsed sections** (`<details><summary>`): Detailed installation steps, changelogs, large config blocks
7. **Task lists** (`- [ ]` / `- [x]`): Roadmap, feature status

**Optional (marked experimental)**:
8. **Mermaid diagrams**: Only generated when project has >= 3 modules, using simple `graph TD` syntax, with `<!-- Experimental: if rendering fails, preview on GitHub -->` comment. Avoid complex structures like `flowchart`, `subgraph`.
</GitHub_Extended_Syntax>

<Interview_Questions>
The interview must ask only one question at a time. Each question provides contextual options based on code analysis results.

**Q1: Project Introduction & Positioning**
Based on the tech stack discovered in code analysis, ask: What does this project do? What is its one-line pitch?

**Q2: Target Audience**
Ask: Who will use this project? (Developers / End Users / Operations / Other Teams)

**Q3: Installation & Run Instructions**
Based on the package manager identified in code analysis (npm/pip/cargo/go mod), suggest candidate installation commands and ask user to confirm or modify.
→ Triggers code recheck: Use Read to verify that dependency names in package.json/requirements.txt/Cargo.toml and script commands match user's description.

**Q4: Usage Examples**
Ask: Please provide 1-2 typical usage scenarios.
→ Triggers code recheck: Use Read to verify that API signatures and CLI entry arguments match user's examples.

**Q5: Multilingual Requirements**
Ask: Which languages do you need the README in? (Default: source language and English)
Options: Chinese, English, Japanese, Korean, French, German, etc. Maximum 5 languages.

**Gap Check (maximum 2 rounds)**
After Q1-Q5, check for contradictions based on user answers:
- User-claimed API functions that don't exist in actual code → flag and follow up
- User installation commands that don't match code's package manager → flag and follow up
- User-mentioned config keys inconsistent with actual config files → flag and follow up
- Unresolvable contradictions marked as "Pending user review" and preserved in checkpoint
</Interview_Questions>

<Checkpoint_Format>
Write checkpoint file `.readme-checkpoint-{N}.md` after each Phase, format:

```markdown
# README Checkpoint {N}: {Phase Name}
> Time: {timestamp}
> Project: {project_path}
> Size: {small|medium|large|empty}

## Extracted Data
{phase-specific data}

## Decision Log
{key decisions made in this phase}

## Open Issues
{unresolved issues for next phase}
```

If interrupted and resumed, first check for checkpoint files, continue from the Phase after the last completed one.
</Checkpoint_Format>

<Steps>

## Phase 0: Security Gate + Existing README Analysis

**Goal**: Establish security boundaries, check existing documentation, determine project size.

**Execution Steps**:

1. **Check existing README**:
   - Glob for `README.md`, `readme.md`, `README.MD` in the current project directory
   - If found, Read the full content, analyze existing structure, identify retainable sections (narrative content such as introduction, usage scenarios, contributor list, acknowledgments)
   - Record which sections are user narrative (retain) and which are technical facts (will be overwritten by code analysis)

2. **Establish file exclusion list**:
   - Add all patterns listed in Secrets_Exclusion to the exclusion list
   - Check all subsequent Glob/Read operations against the exclusion list

3. **Determine project size**:
   - Count source files using Glob:
   ```
   **/*.{py,js,ts,jsx,tsx,go,rs,java,rb,php,c,cpp,h,swift,kt,cs,scala,r,ex,exs}
   ```
   - Exclude `node_modules`, `.git`, `dist`, `build`, `vendor`, `target`, `.next`, `__pycache__`, `.venv`, `venv`
   - Determine size tier per Context_Budget table

4. **Write checkpoint**:
   - Write `.readme-checkpoint-0.md`, containing: existing README analysis results, project size tier, exclusion file list

## Phase 1: Code Analysis (Token Budgeted)

**Goal**: Extract technical facts from code, establish ground truth.

**Execution Steps**:

1. **Identify project type and build system**:
   - Glob for `package.json`, `pyproject.toml`, `setup.py`, `requirements.txt`, `Cargo.toml`, `go.mod`, `Gemfile`, `pom.xml`, `build.gradle*`, `CMakeLists.txt`, `Makefile`
   - Read found manifest files, extract: project name, version, dependencies, script commands
   - Replace all values with placeholders (e.g., API key → `<YOUR_API_KEY>`)

2. **Identify entry files**:
   - Look for typical entry files by project type: `index.{js,ts}`, `main.{py,go,rs}`, `app.{py,js,ts}`, `src/index.*`, `src/main.*`, `src/app.*`
   - Read entry files (counts toward file read quota), extract: primary function/class/API endpoint signatures

3. **Identify license**:
   - Glob for `LICENSE`, `LICENSE.*`, `LICENCE`, `COPYING`
   - Read the first line of license file, identify license type (MIT, Apache-2.0, GPL, BSD, etc.)

4. **Deep analysis by size strategy** (per size determined in Phase 0):
   - **Small**: Read project directory tree, key source files under src/ (up to 15 total files), all config files
   - **Medium**: Read entry files + top-level config + file lists from 2-3 key subdirectories under src/ (up to 10 total files)
   - **Large**: Read package manifest + top-level config + 1 entry file (up to 6 total files)
   - **Empty**: Skip code analysis, mark "No code detected"

5. **Extract structured data**:
   - Language / Framework / Runtime
   - Package manager
   - Entry file
   - Public API signatures
   - Config key names (values replaced with placeholders)
   - License type
   - Project directory structure

6. **Write checkpoint**:
   - Write `.readme-checkpoint-1.md`, containing tech_stack, apis, configs, structure, license, package_manager

## Phase 2: Bidirectional Interview (with Gap Check)

**Goal**: Collect user intent and verify user claims against code through code rechecking.

**Execution Steps**:

**Mode Check**: First determine `--mode` parameter:
- If `--mode direct`: Skip all interview steps below, proceed directly to **Direct Mode Auto-Inference** (Step 8)
- If `--mode interview` (default): Execute steps 1-7 in order

---

### Interview Mode Steps:

1. **Q1: Project Introduction & Positioning**
   - Based on Phase 1 tech_stack, ask what the project does
   - Use AskUserQuestion tool, providing context-aware options based on code analysis

2. **Q2: Target Audience**
   - Ask about target user groups

3. **Q3: Installation & Run Instructions**
   - Based on the package manager identified in Phase 1, suggest candidate installation commands
   - After user response, trigger code recheck:
     - Read dependency manifest file, verify referenced dependency names exist
     - Read package.json scripts / Makefile targets, verify referenced run commands are valid
   - On mismatch: notify user, code is authoritative

4. **Q4: Usage Examples**
   - Ask for 1-2 typical usage scenarios
   - After user response, trigger code recheck:
     - Read entry file or API file, verify function/endpoint signatures mentioned by user
     - On mismatch: notify user, code is authoritative

5. **Q5: Multilingual Requirements**
   - AskUserQuestion: Which languages are needed for the README?
   - Provide common language options (Chinese, English, Japanese, Korean, French, German, etc.)
   - Default includes at least source language and English; maximum 5

6. **Gap Check (maximum 2 rounds)**:
   - Synthesize Q1-Q5 user answers against Phase 1 code facts
   - Find contradictions: API functions not found, config key mismatches, install command mismatches with package manager
   - For each contradiction, Read the relevant source file to confirm, then follow up with user for clarification
   - If unresolved after round 1, proceed to round 2
   - If still unresolved after 2 rounds, mark as "Pending user review" and preserve

7. **Write checkpoint**:
   - Write `.readme-checkpoint-2.md`, containing purpose, audience, install_steps, usage_examples, languages, gap_check_results

---

### Direct Mode Auto-Inference (Skip Interview):

8. **Auto-infer project narrative** (only `--mode direct`):

   **Project Description**:
   - Extract from `package.json` `description` field
   - Or from `pyproject.toml` `[project] description`
   - Or from `Cargo.toml` `[package] description`
   - If none found, extract from top comment in entry file
   - If still none, use directory name as project name + "A software project" placeholder description
   - Tag as `[Auto-generated from code analysis]`

   **Target Audience**:
   - Default "Developers"

   **Installation Commands**:
   - npm project: `npm install` + `npm run <start-script>`
   - pip project: `pip install -e .` or `pip install -r requirements.txt`
   - cargo project: `cargo build --release` + `cargo run`
   - go project: `go build ./...` + `go run .`
   - All tagged `[Auto-generated]`

   **Usage Examples**:
   - Generate 1-2 minimal code examples from Phase 1 API signatures
   - CLI projects: generate `--help` style usage examples
   - Library projects: generate import + minimal invocation examples
   - All tagged `[Auto-generated]`

   **Language**:
   - Source language only (no translation), marked as single-language output

9. **Write checkpoint**:
   - Write `.readme-checkpoint-2.md`, containing auto_inferred purpose, audience, install_steps, usage_examples, languages=single, tagged `mode: direct`

## Phase 3: README Generation + Cross-Validation

**Goal**: Generate a README draft and cross-validate against source code to ensure accuracy.

**Execution Steps**:

1. **Build README structure**:
   Required sections (in order):
   - Title (with badges)
   - Table of Contents (anchor links)
   - Introduction
   - Installation
   - Usage
   - API Reference
   - Contributing
   - License

2. **Apply GitHub extended syntax**:
   - Badges: Tests, License, Version, Language — using `<div align="center"><img alt="...">...</div>` format. All badge `<img>` tags MUST be inside a centered div, placed inline with no blank lines between them to render horizontally.
   - Language switcher (when >= 2 languages): A `<p align="center">` block right after badges (before TOC), containing flag emoji + links to all language versions. Example: `<p align="center">🇺🇸 <a href="./README.md">English</a> | 🇨🇳 <a href="./README.zh.md">简体中文</a></p>`
   - Code blocks: all code examples with language annotation
   - Tables: API reference, configuration options presented as tables
   - Anchor links: each section in TOC with anchors
   - Alerts: use `> [!NOTE]` / `> [!WARNING]` / `> [!IMPORTANT]` / `> [!CAUTION]` in appropriate places
   - Collapsed sections: wrap detailed install steps and large config blocks with `<details><summary>`
   - Task lists: roadmap items with `- [ ]` / `- [x]`
   - Mermaid diagrams: only when >= 3 modules (marked experimental)

3. **Cross-validation (critical step)**:
   - Extract all technical claims from the generated README (API signatures, config keys, CLI commands, dependency names)
   - Re-read up to 5 key source files, verify each item
   - On mismatch: correct the README claim
   - After cross-validation completes, record validation results

4. **Write checkpoint**:
   - Write `.readme-checkpoint-3.md`, containing full README draft + cross_check_report

## Phase 4: User Review Gate

**Goal**: User reviews the README draft, confirms before final output.

**Execution Steps**:

1. **Present README draft**:
   - Show user the complete README generated in Phase 3
   - Annotate content sources: which come from code analysis (tag [Code]), which from interview (tag [Interview]), which are auto-inferred (tag [Auto-generated], direct mode only)

2. **Provide structural completeness report**:
   - List whether all required sections exist
   - GitHub syntax usage checklist

3. **User decision**:
   - User says "done" / "ok" / "confirm" / "approved" → proceed to Phase 5
   - User requests changes → adjust README based on feedback, return to end of Phase 3 to regenerate draft
   - Maximum 3 revision rounds
   - If still unsatisfied after 3 rounds: save current draft as `README.draft.md`, suggest manual editing

4. **Overwrite strategy**:
   - If `README.md` already exists: AskUserQuestion whether to overwrite or save as `README.new.md`
   - If not: mark that `README.md` will be written

5. **Note**: This Phase does not write a checkpoint file; wait for user confirmation.

## Phase 5: Multilingual Translation

**Goal**: Translate the final README into target languages.

**Mode Check**:
- If `--mode direct`: Skip translation, proceed directly to Phase 6. Direct mode only generates source-language README.
- If `--mode interview` (default): Execute translation per steps below.

**Execution Steps**:

1. **Prerequisite**: Phase 4 user confirmed + non-direct mode

2. **Translation rules**:
   - Preserve all code block content unchanged
   - Preserve all API names, function signatures, class names unchanged
   - Only translate descriptive text (headings, paragraphs, text descriptions in tables)
   - Badge URL label text adjusted per language
   - Alert block titles translated (e.g., `> [!NOTE]` → localized equivalent)
   - Mermaid diagram node labels translated

3. **Language switcher (MANDATORY for >= 2 languages)**:
   - Insert a `<p align="center">` language switcher in EVERY translated README, positioned after the badges and before the TOC
   - Use flag emojis: 🇺🇸 English, 🇨🇳 简体中文, 🇯🇵 日本語, 🇰🇷 한국어, 🇫🇷 Français, 🇩🇪 Deutsch
   - Link each flag to the corresponding `README.{lang}.md` (or `README.md` for the default language)
   - Example for zh,en output:
     ```html
     <p align="center">
     🇺🇸 <a href="./README.md">English</a> | 🇨🇳 <a href="./README.zh.md">简体中文</a>
     </p>
     ```
   - The language switcher content itself is NOT translated — it stays identical across all language versions

4. **Processing flow**:
   - If target languages > 2, process each language sequentially
   - Write `README.{lang}.md` after each language translation completes
   - Save progress between each language translation

4. **Optional localization notes**:
   - Confirm with user during interview whether to add region-specific links (community links, mirrors, etc.)

5. **Write checkpoint**:
   - Write `.readme-checkpoint-5.md`, listing all translated language versions and file paths

## Phase 6: Final Output

**Goal**: Write final files and clean up.

**Execution Steps**:

1. **Write main README**:
   - Based on Phase 4 user choice, write `README.md` (overwrite mode) or `README.new.md` (preserve original mode)

2. **Write multilingual versions**:
   - Write `README.{lang}.md` for each target language (e.g., `README.zh.md`, `README.ja.md`)

3. **Clean up checkpoint files**:
   - Delete `.readme-checkpoint-0.md`
   - Delete `.readme-checkpoint-1.md`
   - Delete `.readme-checkpoint-2.md`
   - Delete `.readme-checkpoint-3.md`
   - Delete `.readme-checkpoint-5.md`
   - (Phase 4 has no checkpoint, no cleanup needed)

4. **Output summary**:
   - List all generated files and paths
   - Report GitHub syntax usage stats (badge count, table count, alert types, etc.)
   - Report cross-validation results (how many items checked, how many corrected)
   - Remind user of any "Pending user review" contradictions

</Steps>

<Integrated_Quality_Check>
The following quality check results must be included in the output summary after generation:

### Structural Completeness (Required Items Check)
- [ ] Title with horizontally centered badges (`<div align="center">`)
- [ ] Language switcher (MANDATORY if >= 2 languages; `<p align="center">` with flag emoji links to all language versions)
- [ ] TOC with anchor links
- [ ] Introduction section
- [ ] Installation section
- [ ] Usage section
- [ ] API Reference section (if applicable)
- [ ] Contributing section
- [ ] License section

### Code Consistency (Cross-Validation)
- [ ] Phase 3 cross-validation completed
- [ ] API signatures match source code
- [ ] Config key names match source code
- [ ] Installation commands match package manager

### GitHub Syntax Usage
- [ ] Badges (`<div align="center"><img alt="..."></div>` — centered, horizontal)
- [ ] Language switcher (MANDATORY if >= 2 languages; `<p align="center">` with flag emoji links)
- [ ] Code blocks (with language annotation)
- [ ] Tables
- [ ] Anchor links
- [ ] Alerts (if applicable)
- [ ] Collapsed sections (if applicable)
- [ ] Task lists (if applicable)
- [ ] Mermaid diagrams (if applicable, marked experimental)

### Security Check
- [ ] No keys/tokens/passwords exposed
- [ ] No internal endpoints exposed
- [ ] All configuration values are placeholders
</Integrated_Quality_Check>

<Recovery_Procedure>
If execution is interrupted (timeout, network error, user cancellation, etc.):

1. When re-invoking this skill, first check for `.readme-checkpoint-*.md` files
2. If checkpoint files exist, read the latest checkpoint
3. Resume from the Phase after the last completed one
4. "Open Issues" in checkpoints are prioritized on resume
5. If Phase 1 code analysis checkpoint exists, skip Phase 0-1, start directly from Phase 2
6. If all checkpoints exist and Phase 4 was confirmed, start from Phase 5
</Recovery_Procedure>

<Edge_Cases>
- **Empty project**: Phase 1 skips code analysis. Interview mode: Phase 2 relies entirely on interview. Direct mode: generates minimal placeholder README, tagged "⚠️ This project contains no detectable code yet. Please update the README after actual development."
- **No package manager file**: Skip install command auto-inference, ask user directly in Phase 2 Q3 how to install
- **No LICENSE file**: Ask user in Phase 2 about intended license, tag license section as "To be confirmed"
- **Existing README seriously inconsistent with code**: Flag in Phase 0, highlight conflicting content in Phase 4 presentation
- **User skips interview questions**: If user declines to answer a question, fill that section with code analysis results, tagged `[Auto-generated from code analysis]`
</Edge_Cases>
