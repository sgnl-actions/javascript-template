# SGNL Action Template

Starting point for building JavaScript actions that run on the SGNL CAEP Hub.

## Creating a new action

```bash
gh repo create sgnl-actions/my-action --public --template sgnl-actions/javascript-template --clone
cd my-action
npm install
```

Then edit `src/script.mjs` with your logic, update `metadata.yaml` with your inputs/outputs, and write tests in `tests/script.test.js`.

## How actions work

Actions are stateless Node.js scripts that the CAEP Hub fetches from GitHub and runs in isolated worker containers. Each action exports a default object with up to three handlers:

```javascript
export default {
  invoke: async (params, context) => { /* required — your main logic */ },
  error:  async (params, context) => { /* optional — recovery on failure */ },
  halt:   async (params, context) => { /* optional — cleanup on cancellation */ }
};
```

**`invoke`** receives the job's input parameters and a context object with environment variables, secrets, and outputs from previous workflow steps. It returns an object whose keys must match the outputs declared in `metadata.yaml`.

**`error`** is called when `invoke` throws. If it returns successfully, the error is considered recovered. If it re-throws, the error is fatal and no retry happens. If you don't define an error handler, all errors are retryable by default.

**`halt`** is called when a job is cancelled or times out. Use it to clean up resources.

### The context object

```javascript
{
  env: { ENVIRONMENT: "production", ... },
  secrets: { API_TOKEN: "...", ... },
  outputs: { "previous-step": { user_id: "123", ... } }
}
```

Secrets are never hardcoded — they come from `context.secrets`. Environment variables come from `context.env`.

## File structure

| File | Purpose |
|------|---------|
| `src/script.mjs` | Your action logic (ES6 modules) |
| `tests/script.test.js` | Jest tests |
| `metadata.yaml` | Declares inputs, outputs, secrets, and environment variables |
| `dist/index.js` | Built CommonJS bundle (generated, committed) |
| `dist/licenses.txt` | Third-party license notices (generated, committed) |
| `rollup.config.mjs` | Build config — extends `@sgnl-actions/rollup-config` |
| `eslint.config.mjs` | Lint config — re-exports `@sgnl-actions/eslint-config` |
| `scripts/dev-runner.js` | Local testing with mock data |

## metadata.yaml

This file defines your action's interface. It gets validated against a [JSON Schema](https://github.com/sgnl-actions/schemas/blob/main/metadata.schema.json) during CI, and the conformance checker verifies that your code's inputs and outputs actually match what you declared.

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/sgnl-actions/schemas/main/metadata.schema.json
name: my-action
description: What this action does

inputs:
  user_email:
    type: text
    description: Email of the user to process
    required: true
    validation:
      min: 5
      max: 254

outputs:
  status:
    type: text
    description: Result of the operation
```

Add the `yaml-language-server` comment at the top and your editor will give you autocomplete and validation (VS Code needs the [YAML extension](https://marketplace.visualstudio.com/items?itemName=redhat.vscode-yaml) by Red Hat).

Input types: `text`, `number`, `boolean`, `array`, `email`. Output types add `datetime` and `object`. Validation supports `min`, `max`, `regex`, and `enum`.

## Development

```bash
npm run dev            # run with mock data from scripts/dev-runner.js
npm test               # run tests
npm run test:coverage  # run tests with coverage (80% minimum)
npm run lint           # check code style
npm run lint:fix       # auto-fix style issues
npm run validate       # check metadata schema + code conformance
npm run build          # bundle to dist/index.js + generate dist/licenses.txt
```

### Build process

`npm run build` does two things:

1. **Rollup** bundles `src/script.mjs` into `dist/index.js` as CommonJS (the worker runtime expects `module.exports`, not ES6 `export`). The bundle is minified with terser, and the MIT license banner is preserved at the top.

2. **generate-license-file** scans your production dependencies and writes `dist/licenses.txt` with the full license text of every bundled library. This is required for OSS compliance since the bundle is distributed publicly via GitHub.

Both output files are committed to the repo. CI verifies they're up to date — if you change source code and forget to rebuild, the PR will fail.

### Shared configuration

Most configuration is centralized in shared packages so you don't have to maintain it per-repo:

| Package | What it provides |
|---------|-----------------|
| [`@sgnl-actions/eslint-config`](https://github.com/sgnl-actions/eslint-config) | ESLint 10 flat config with org-standard rules |
| [`@sgnl-actions/rollup-config`](https://github.com/sgnl-actions/rollup-config) | Rollup config with terser, MIT banner, OSS comment preservation |
| [`@sgnl-actions/validate`](https://github.com/sgnl-actions/validate) | `sgnl-validate` CLI for metadata schema + code conformance checking |
| [`sgnl-actions/.github`](https://github.com/sgnl-actions/.github) | Reusable CI and Release GitHub Actions workflows |
| [`sgnl-actions/schemas`](https://github.com/sgnl-actions/schemas) | JSON Schema for `metadata.yaml` |

If your action needs custom rollup plugins (e.g., `@rollup/plugin-json` for AWS SDK), extend the base config:

```javascript
import { createConfig } from '@sgnl-actions/rollup-config';
import json from '@rollup/plugin-json';

export default createConfig({
  inlineDynamicImports: true,
  plugins: [json()]
});
```

## Testing

Tests use Jest. Mock external API calls — actions should be testable without network access.

```javascript
import script from '../src/script.mjs';

const mockContext = {
  env: { ENVIRONMENT: 'test' },
  secrets: { API_TOKEN: 'test-token' },
  outputs: {}
};

test('should do the thing', async () => {
  const result = await script.invoke({ user_email: 'test@example.com' }, mockContext);
  expect(result.status).toBe('success');
});
```

The conformance checker (`npm run validate`) parses your source code and verifies that the `invoke` handler reads the inputs declared in `metadata.yaml` and returns the declared outputs. Return values should be explicit object literals (not function calls) so the checker can verify them statically.

## CI

Every push and pull request triggers the CI workflow (defined in `sgnl-actions/.github`). It runs these steps in order:

1. `npm ci` — install dependencies
2. **Package-lock sync check** — fails if `package-lock.json` is out of date with `package.json`
3. `npm run validate` — metadata schema validation + code conformance check
4. `npm run lint` — ESLint
5. `npm test` — Jest
6. `npm run build` — Rollup bundle + license file generation
7. **Dist freshness check** — fails if `dist/index.js` or `dist/licenses.txt` differ from what's committed

If any step fails, the PR is blocked. The most common failure is forgetting to run `npm run build` after changing source code.

## Releases

Releases are version-tagged snapshots of the repo.

To create a release:

1. Make sure all your changes are merged to `main`
2. Go to the repo's **Actions** tab
3. Select the **Release** workflow
4. Click **Run workflow**
5. Pick the bump level:
   - **patch** (v1.0.0 -> v1.0.1) — bug fixes, dependency updates
   - **minor** (v1.0.0 -> v1.1.0) — new features, non-breaking changes
   - **major** (v1.0.0 -> v2.0.0) — breaking changes to inputs, outputs, or behavior

The workflow runs the full CI pipeline first. If it passes, it computes the next version from the latest git tag, creates an annotated tag on the current `main` HEAD, and publishes a GitHub Release with auto-generated notes from merged PRs.

No commits are pushed to `main` during a release — the tag points to the existing HEAD. This avoids branch protection conflicts.

Reference your action by tag in job requests:

```json
{
  "script": {
    "repository": "github.com/sgnl-actions/my-action@v1.2.0",
    "type": "nodejs"
  }
}
```

## Dependabot

Each repo has Dependabot configured to open weekly PRs for npm and GitHub Actions dependency updates. Related packages are grouped (e.g., all rollup plugins in one PR, all eslint packages in one PR) to reduce noise.

## Implementation checklist

When creating a new action from this template:

- [ ] Update `name` and `description` in `metadata.yaml` (name must be kebab-case)
- [ ] Define inputs with types, descriptions, and validation rules
- [ ] Define outputs with types and descriptions
- [ ] Add secrets and environment variables if your action calls external APIs
- [ ] Implement `invoke` handler in `src/script.mjs`
- [ ] Add an `error` handler if you need custom retry logic or fatal error detection
- [ ] Write tests covering success, validation errors, and API failures
- [ ] Update `scripts/dev-runner.js` with realistic mock parameters
- [ ] Run `npm run validate && npm run lint && npm test && npm run build`
- [ ] Update this README with your action's specific documentation
