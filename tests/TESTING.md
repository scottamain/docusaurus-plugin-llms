# Testing the docusaurus-plugin-llms

## Running Tests

Run the full test suite (build + unit + integration):

```bash
npm test
```

Or run individual stages:

```bash
npm run build            # Compile TypeScript
npm run test:unit        # Unit tests only
npm run test:integration # Integration tests only
```

### Standalone Test Files

Some test files are not included in `npm run test:unit` but can be run directly:

```bash
node tests/test-route-resolution-helpers.js   # Route resolution integration tests
node tests/test-refactored-route-helpers.js   # Suffix-matching unit tests
node tests/test-numbered-prefixes.js          # Numbered prefix handling
```

## Test Categories

### Route Resolution (`test-route-resolution-helpers.js`)

Integration tests that exercise `processFilesWithPatterns` end-to-end with
real files on disk. Covers:

- **Suffix-based matching** — files resolve to routes via `routesPaths`
- **Numbered prefix stripping** — `01-intro.md` matches `/intro`
- **docsDir stripping** (Issue #31) — prevents doubled `docs/docs/` paths
  when `routeBasePath: '/'`
- **Trailing-slash routes** (Issue #30) — routes like `/intro/` match correctly
- **Directory collapsing** — `generics/generics.md` resolves to `/generics`
- **Frontmatter overrides** — files with `id` or `slug` in frontmatter
  resolve correctly when the filename differs from the route
- **Versioned route disambiguation** — shortest route (stable) preferred
  over versioned (`/nightly/...`)
- **Fallback** — graceful path-based URL construction when `routesPaths`
  is not available

### Route Helpers (`test-refactored-route-helpers.js`)

Unit tests for the individual helper functions used by route resolution:

- `findMatchingRoute` — suffix matching, edge cases, shortest-match preference
- `collapseMatchingTrailingSegment` — directory index collapsing
- `removeNumberedPrefixes` — `01-category/02-file` → `category/file`
- Path normalization — extension removal, index handling, Windows paths
- URL construction — `new URL()` with various `siteUrl` formats

### Numbered Prefixes (`test-numbered-prefixes.js`)

Focused tests for numbered prefix scenarios: exact match priority,
fallback to prefix removal, multi-level nesting, mixed segments,
trailing slashes, and versioned route disambiguation.

### Path Transformation (`test-path-transforms.js`, `test-path-transformation.js`)

Tests for the `pathTransformation` option (`ignorePaths`, `addPaths`).

### Other Tests

The `test:unit` script runs ~27 additional test files covering:

- Plugin option validation and input sanitization
- Markdown processing (import removal, header deduplication, partials)
- File I/O error handling, BOM detection, circular import detection
- URL encoding, filename sanitization, regex safety
- Batch/parallel processing, path length validation

## Testing in a Real Docusaurus Project

To test the plugin in a real Docusaurus project:

1. **Link or reference your local plugin**:

   In your Docusaurus project's `package.json`:
   ```json
   "docusaurus-plugin-llms": "file:/path/to/docusaurus-plugin-llms"
   ```

   Then run `npm install`.

2. **Add the plugin to `docusaurus.config.js`**:

   ```js
   plugins: [
     [
       'docusaurus-plugin-llms',
       {
         title: 'My Documentation',
         description: 'Description of the docs.',
         excludeImports: true,
         removeDuplicateHeadings: true,
       },
     ],
   ],
   ```

3. **Build and inspect**:

   ```bash
   npm run build
   ```

   Check `build/llms.txt` and verify that URLs match your site's actual routes.

## How Route Resolution Works

The plugin resolves file paths to URLs using suffix-based matching against
Docusaurus's `routesPaths` (provided in the `postBuild` hook). This avoids
needing to know about `routeBasePath`, version prefixes, or `baseUrl` — any
route ending with the file's path tail is a match.

Resolution order:
1. Strip `docsDir` prefix and file extension to get a "tail"
2. Try suffix match against `routesPaths` (original tail)
3. Try directory-collapsed tail (`generics/generics` → `generics`)
4. Try numbered-prefix-stripped tail (`01-intro` → `intro`)
5. Read frontmatter `slug`/`id` and retry suffix match
6. Fall back to path-based URL construction
