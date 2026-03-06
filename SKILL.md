name: Intent Compiler
description: Translates natural language specifications into test cases, documentation, code skeletons, and acceptance criteria
tags: [nlp, testing, documentation, prototyping, ai-assisted, code-generation]
version: 1.2.0
author: OpenClaw Team
requires:
  - python>=3.9
  - openai>=1.0.0
  - jinja2>=3.0.0
  - pyyaml>=6.0
  - git
env:
  OPENCLAW_INTENT_MODEL: "gpt-4"
  OPENCLAW_INTENT_TEMP: "0.3"
  OPENCLAW_INTENT_MAX_TOKENS: "2000"
```

# Intent Compiler

## Purpose

The Intent Compiler transforms unstructured natural language specifications into concrete, executable artifacts. It bridges the gap between high-level requirements and implementation-ready outputs.

### Real Use Cases

- **Test Generation**: Convert "As a user, I want to login with email and password" into Jest/Pytest test skeletons with assertions
- **Documentation Production**: Transform function docstrings and usage patterns into comprehensive README.md, API reference docs, or changelog entries
- **Code Scaffolding**: Generate TypeScript interfaces, React component skeletons, or Express routes from endpoint descriptions
- **Acceptance Criteria Definition**: Parse Gherkin-like feature descriptions into structured scenarios
- **Backward Compatibility Checking**: Generate test matrices to validate API contract adherence
- **Migration Plan Synthesis**: Create step-by-step migration guides from legacy to new architecture

## Scope

### Primary Commands

```
intent-compiler generate \
  --type <test|docs|scaffold|criteria|migration> \
  --input <file|string> \
  --output <path> \
  [--template <name>] \
  [--context <dir>] \
  [--dry-run]

intent-compiler validate \
  --spec <file> \
  --against <commit|branch>

intent-compiler interactive \
  --mode <refinement|iteration> \
  --seed <initial-prompt>

intent-compiler suggest \
  --based-on <error-output|test-failure> \
  --context <file://path>
```

### CLI Flags

- `--type`: Artifact category (required)
- `--input`: Source spec - can be plaintext, markdown, or YAML feature file
- `--output`: Destination path; uses templates if directory
- `--template`: Jinja2 template name (default: `default_<type>.j2`)
- `--context`: Additional files/directories for RAG context
- `--dry-run`: Preview without writing files
- `--based-on`: For `suggest` command - failing test output
- `--mode`: Interactive refinement loop (`refinement` for edit-suggest cycles, `iteration` for multiple variations)

## Detailed Work Process

### 1. Intent Parsing
- Load input from file or stdin
- Detect intent type if not provided using classifier (test/docs/scaffold/criteria)
- Extract domain entities (e.g., "user", "order", "payment")
- Identify constraints (async, validation, permissions)

### 2. Context Assembly
- If `--context` provided, scan files for related code patterns
- Build prompt with:
  - Original spec
  - Detected tech stack from `package.json`, `requirements.txt`, or `Cargo.toml`
  - Existing test examples if present
  - Style guide snippets if found

### 3. Generation Phase
- Construct OpenAI prompt with system message templates from `templates/system_<type>.txt`
- Call LLM with temperature from env (default 0.3 for deterministic outputs)
- Stream response into structured intermediate format (JSON) before templating
- Apply Jinja2 template to produce final artifact

### 4. Validation & Linting
- For tests: check for proper imports, test naming conventions
- For docs: verify heading hierarchy, code block syntax
- For scaffold: ensure no syntax errors in generated code (basic parse check)
- Output warnings to stderr, return exit code 1 if critical failures

### 5. Output Writing
- Create parent directories if needed
- Write file with 644 permissions
- If output is directory: generate multiple scaffold files (e.g., `test/<component>.spec.ts`, `docs/api/<module>.md`)

### Workflow Example (Test Generation)

```bash
# User writes spec in file 'feature.md':
# "Login endpoint accepts POST /auth/login with email and password, returns JWT on success"

intent-compiler generate \
  --type test \
  --input feature.md \
  --output tests/auth/ \
  --template react-query
```

Process:
1. Parse: identifies `POST /auth/login`, parameters `email`, `password`, response `JWT`
2. Context: finds `package.json` with `@testing-library/react` and `msw`
3. Generate: LLM produces Jest + React Testing Library test with mock service worker
4. Validate: checks import paths, test naming (`describe('auth login', ...)`)
5. Write: `tests/auth/login.test.tsx`

## Golden Rules

### Must-Follow

1. **Atomic Prompts**: Each generation must target a single intent type; never mix test + docs in one call
2. **Template Versioning**: Store templates in `~/.openclaw/templates/` with semantic versioning; update requires manual migration
3. **Context Limiting**: Maximum 5 context files per run to avoid token overflow; use `--context-dir` with `.intentignore` patterns
4. **Review Gate**: Never write directly to production directories (e.g., `src/`, `app/`); default output should be `generated/` or `tmp/`
5. **Idempotency**: Re-running with same input should produce bitwise-identical output when `--temp=0`
6. **Safety**: Never generate destructive operations (rm -rf, drop database) without explicit `--force` flag and human review
7. **Attribution**: Include comment header with generation timestamp, input hash, and template version

### Anti-Patterns to Avoid

- Don't generate entire microservices from one-sentence specs
- Don't auto-commit generated code; always leave to user
- Don't infer business logic; stick to structural patterns
- Avoid hard-coded values; use placeholders like `{{API_BASE_URL}}`
- Never expose API keys in generated examples

## Examples

### Example 1: Test Skeleton Generation

**Input** (`specs/search.feature.md`):
```markdown
Feature: Product search
  As a shopper
  I want to search products by name
  So that I can find items quickly

  Scenario: Search returns matching products
    Given the catalog contains "widget", "gadget", "doohickey"
    When I search for "wid"
    Then I see "widget" in results
```

**Command**:
```bash
intent-compiler generate \
  --type test \
  --input specs/search.feature.md \
  --output tests/unit/product_search.test.ts \
  --template cucumber-ts
```

**Output** (`tests/unit/product_search.test.ts`):
```typescript
// Generated by Intent Compiler v1.2.0 | Template: cucumber-ts
// Input: specs/search.feature.md (sha256: a3f8...)
// DO NOT EDIT MANUALLY - regenerate via intent-compiler

import { test, expect } from '@playwright/test';

test.describe('Product search', () => {
  test('Search returns matching products', async ({ page }) => {
    // Given the catalog contains "widget", "gadget", "doohickey"
    await page.goto('/admin/seed?items=widget,gadget,doohickey');

    // When I search for "wid"
    await page.fill('[data-testid=search-input]', 'wid');
    await page.click('[data-testid=search-submit]');

    // Then I see "widget" in results
    await expect(page.locator('[data-testid=product-card]')
      .filter({ hasText: 'widget' })).toBeVisible();
  });
});
```

### Example 2: API Documentation Generation

**Input** (`src/api/v1/users.ts`):
```typescript
/**
 * GET /api/v1/users
 * Query: ?role=admin&page=1
 * Returns paginated user list
 */
export async function listUsers(req: Request) { ... }

/**
 * POST /api/v1/users
 * Body: { email: string, name: string }
 * Returns created user with id
 */
export async function createUser(req: Request) { ... }
```

**Command**:
```bash
intent-compiler generate \
  --type docs \
  --input src/api/v1/users.ts \
  --output docs/api/v1/users.md \
  --template openapi-json
```

**Output** (`docs/api/v1/users.md`):
```markdown
# Users API (v1)

Base URL: `/api/v1`

## List Users

`GET /users`

Query Parameters:
| Name  | Type   | Required | Description           |
|-------|--------|----------|-----------------------|
| role  | string | false    | Filter by role        |
| page  | number | false    | Page number (default: 1) |

Response (200):
```json
{
  "data": [{ "id": "uuid", "email": "...", "name": "..." }],
  "page": 1,
  "total": 150
}
```

## Create User

`POST /users`

Request Body:
```json
{
  "email": "string",
  "name": "string"
}
```

Response (201):
```json
{
  "id": "uuid",
  "email": "...",
  "name": "..."
}
```
```

### Example 3: Code Scaffolding with Context

**Command**:
```bash
intent-compiler generate \
  --type scaffold \
  --input "Create a Next.js 14 page for /dashboard/settings with form fields: theme, notifications, language" \
  --output app/dashboard/settings/page.tsx \
  --context app/(auth)/layout.tsx,components/ui/Form.tsx \
  --template next14-page
```

**Generated Output**: Includes `useServerAction` pattern, imports from existing Form component, respects layout structure.

## Rollback Commands

### File-Level Rollback

```bash
# Undo last generation if git-tracked
git checkout HEAD -- app/dashboard/settings/page.tsx

# Or restore from backup timestamp
cp ~/.openclaw/backups/20240315_143022/app/dashboard/settings/page.tsx \
   app/dashboard/settings/page.tsx
```

### Session-Level Cleanup

```bash
# Remove all generated files from current session
intent-compiler --cleanup-temp --session-id $(cat ~/.openclaw/last_session_id)
```

### State Rollback

```bash
# Revert template registry to previous version (if updated)
git -C ~/.openclaw/templates checkout v1.1.0
```

## Verification Steps

### Post-Generation Checks

1. **Syntax Validation**:
   ```bash
   # For TypeScript
   npx tsc --noEmit generated/*.ts 2>&1 | grep -q "error" && echo "FAIL"
   
   # For Python
   python -m py_compile generated/*.py
   ```

2. **Test Dry-Run**:
   ```bash
   # Jest/Vitest
   npx vitest run --reporter=json --silent tests/generated/ > /dev/null
   echo $?  # Should be 0 (tests parse correctly)
   ```

3. **Documentation Link Check**:
   ```bash
   # Verify internal links in generated markdown
   grep -Eo '\[[^]]+\]\([^)]+\)' docs/generated/*.md \
     | cut -d')' -f1 | cut -d'(' -f2 \
     | while read link; do
         if [ ! -f "$link" ] && [[ "$link" != http* ]]; then
           echo "Broken link: $link"
         fi
       done
   ```

4. **Coverage Estimate**:
   ```bash
   # Ensure generated tests exercise required paths
   grep -c "it\|test" tests/generated/*.test.ts | awk '{s+=$1} END {print s" test cases"}'
   ```

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| Empty output file | LLM quota exceeded | Check `OPENAI_API_KEY` limits; add retry `--retries 3` |
| "Template not found" | Missing template file | `ls ~/.openclaw/templates/`; install via `intent-compiler install-template <name>` |
| Garbled unicode | Wrong encoding | Ensure `PYTHONIOENCODING=utf-8`; regenerate with `--encoding utf-8` |
| Tests not discovered | Wrong naming pattern | Match framework: Jest uses `.test.tsx`, PyTest uses `test_*.py` |
| Slow generation | Context too large | Reduce `--context` files; use `.intentignore` to exclude `node_modules`, `dist` |
| Hallucinated imports | Overly creative LLM | Pin template version; set `OPENCLAW_INTENT_TEMP=0.1` for determinism |
| Permission denied | Output dir unwritable | Use `--output ./generated/` inside project; avoid system dirs |

### Debug Mode

```bash
# Show full prompt sent to LLM
OPENCLAW_INTENT_DEBUG=1 intent-compiler generate ... 2>&1 | less

# Trace file writes
strace -e openat,write -f intent-compiler generate ...
```

## Dependencies

- **Runtime**: Python 3.9+, Node.js 18+ (for certain templates)
- **LLM Provider**: OpenAI API (GPT-4 recommended) or Azure OpenAI endpoint
- **Git**: For diff comparisons and rollback tracking
- **Optional**: `jq` (JSON processing in templates), `pandoc` (doc conversion)

### Installation

```bash
# Clone templates repo
git clone https://github.com/openclaw/intent-templates ~/.openclaw/templates

# Install Python package
pipx install intent-compiler

# Configure
echo 'export OPENAI_API_KEY="sk-..."' >> ~/.bashrc
intent-compiler --install-completions  # bash/zsh/fish
```

### Minimal Setup

```bash
mkdir -p ~/.openclaw/templates
intent-compiler verify-env
# Should output: OK: all required templates present
```

## Environment Variables

- `OPENCLAW_INTENT_MODEL`: OpenAI model name (`gpt-4-turbo`, `gpt-3.5-turbo`)
- `OPENCLAW_INTENT_TEMP`: Sampling temperature 0.0-1.0 (default: 0.3)
- `OPENCLAW_INTENT_MAX_TOKENS`: Generation budget (default: 2000)
- `OPENCLAW_INTENT_TEMPLATE_DIR`: Override default template location
- `OPENCLAW_INTENT_DRY_RUN`: Set to `1` to skip file writes globally
- `OPENCLAW_INTENT_LOG_FILE`: Path to JSONL audit log
- `OPENCLAW_INTENT_CONTEXT_LIMIT`: Max context files (default: 5)

```bash
# Example .env
OPENCLAW_INTENT_MODEL="gpt-4-1106-preview"
OPENCLAW_INTENT_TEMP=0.2
OPENCLAW_INTENT_LOG_FILE="$HOME/.openclaw/logs/intent_audit.jsonl"
```

## Performance Tips

- Cache context embeddings: `intent-compiler index-context --dir src/ --db ~/.cache/intent_context.db`
- Use `--type test --template minimal` for faster generation in prototyping
- Batch multiple specs: `find specs/ -name "*.md" -exec intent-compiler generate -i {} -o generated/ \;`
- Reuse intermediate JSON: `--intermediate /tmp/intent_payload.json` to separate generation from templating

## Known Limitations

- LLM may miss edge cases in tests; always review and augment
- Template syntax must match target framework version exactly
- No built-in diff viewer; rely on `git diff` for generated code changes
- Large context (>10 files) may exceed token limits; use selective indexing

## Future Roadmap

- Git integration: `intent-compiler apply --commit` to auto-commit with conventional messages
- Multi-modal: Support for image-based specs (wireframe → scaffold)
- Feedback loop: `intent-compiler improve --from PR-comments` to fine-tune prompts
- Distributed: Worker pool for bulk generation across CI pipelines
```