# External Modules

When a module is marked as external, Rolldown will not bundle it. Instead, the `import` or `require` statement is preserved in the output, and the module is expected to be available at runtime.

```js
// input
import lodash from 'lodash';
console.log(lodash);

// output (lodash is external)
import lodash from 'lodash';
console.log(lodash);
```

This page explains how externals work end-to-end: how a module becomes external, how its import path is determined in the output, and how the relevant options and plugin hooks interact.

## How a Module Becomes External

There are three ways a module can be marked as external:

1. **The [`external`](/reference/InputOptions.external) option** — a config-level pattern (string, regex, array, or function) that tests each import specifier. See the [option reference](/reference/InputOptions.external) for pattern syntax, examples, and caveats.

2. **A plugin's `resolveId` hook** — a plugin can return `{ id, external: true }` (or `"relative"` / `"absolute"`) to explicitly mark a module as external. A plugin can also `return false` to mark the raw specifier as external with the same normalization as the `external` option.

3. **Unresolved modules** — if no plugin or the internal resolver can find a module and the `external` option matches the specifier, Rolldown treats it as external rather than throwing an error.

## The Full Resolution Flow

Here is the step-by-step process Rolldown follows when it encounters an import:

### 1. First `external` check

The raw import specifier (e.g. `'./utils'`, `'lodash'`) is tested against the [`external`](/reference/InputOptions.external) option with `isResolved: false`. If it matches, the specifier is normalized and marked as external immediately — **plugins and the internal resolver are skipped entirely**.

### 2. Plugin `resolveId`

If the first check did not match, plugins get a chance to resolve the import:

| Plugin return value                   | Effect                                                                                                                     |
| ------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `return false`                        | External. Raw specifier is normalized (same path as step 1).                                                               |
| `return { id, external: true }`       | External. Output path depends on [`makeAbsoluteExternalsRelative`](/reference/InputOptions.makeAbsoluteExternalsRelative). |
| `return { id, external: "relative" }` | External. Path is **always** relativized (overrides config).                                                               |
| `return { id, external: "absolute" }` | External. Path is **always** kept verbatim (overrides config).                                                             |
| `return { id }` (no `external`)       | Resolved, continue to step 3 with the resolved ID.                                                                         |
| `return null`                         | No plugin handled it, fall through to step 3.                                                                              |

### 3. Internal resolver

Rolldown's built-in resolver tries to find the module on disk.

### 4. Second `external` check

The resolved ID (e.g. `'/project/node_modules/vue/dist/vue.runtime.esm-bundler.js'`) is tested against the [`external`](/reference/InputOptions.external) option with `isResolved: true`. If it matches, the resolved ID is used **as-is** in the output — no normalization occurs.

### 5. Variant selection

Based on [`makeAbsoluteExternalsRelative`](/reference/InputOptions.makeAbsoluteExternalsRelative) and whether the original specifier was relative, the external module is tagged as `Relative` (re-relativize at render time) or `Absolute` (keep verbatim). Plugin overrides (`"relative"` / `"absolute"`) bypass this step.

## What Path Appears in the Output?

Once a module is external, the import path in the output depends on the specifier type and the [`makeAbsoluteExternalsRelative`](/reference/InputOptions.makeAbsoluteExternalsRelative) option:

- **Bare specifiers** (e.g. `'lodash'`, `'node:fs'`) — appear as-is when matched on the first check. If matched on the second check (resolved path), the full resolved path appears instead (see the [caveat about `/node_modules/`](/reference/InputOptions.external#avoid-node-modules-for-npm-packages)).
- **Relative specifiers** (e.g. `'./utils.js'`) — internally normalized to absolute paths for deduplication, then re-relativized from the output chunk's location at render time (when `makeAbsoluteExternalsRelative` is enabled).
- **Absolute specifiers** (e.g. `'/project/lib/utils.js'`) — behavior depends on `makeAbsoluteExternalsRelative`: converted to relative (`true`), kept absolute (`false`), or conditional on the original specifier (`"ifRelativeSource"`, the default).

See the [`makeAbsoluteExternalsRelative` reference](/reference/InputOptions.makeAbsoluteExternalsRelative) for detailed examples and summary tables of each value.

## Special Cases

### Data URLs

Specifiers with a valid, scoped `data:` URL (e.g. `data:text/javascript,export default 42`) are handled by Rolldown's internal dataurl plugin which **bundles the inline content**. They are not automatically treated as external.

However, `data:` URLs that are not valid or scoped are left unresolved and may be treated as external if they match the `external` option.

### HTTP URLs

Specifiers starting with `http://`, `https://`, or `//` are **automatically treated as external** before the internal resolver runs, regardless of the `external` option. These IDs are emitted as-is and not affected by `makeAbsoluteExternalsRelative`.

```js
import lib from 'https://cdn.example.com/lib.js';
// Always external, emitted as-is
```
