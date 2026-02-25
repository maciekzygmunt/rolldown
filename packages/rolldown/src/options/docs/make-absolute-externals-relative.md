Note that this option does not affect when a relative path is directly marked as "external" using the [`external`](/reference/InputOptions.external) option.

See the [External Modules guide](/in-depth/external-modules) for a detailed explanation of how this option interacts with the `external` option and plugin hooks.

#### Values

##### `"ifRelativeSource"` (default)

Only convert to relative if the **original import specifier** was relative.

```js
// Original: relative specifier → converted to relative in output
import './lib/utils.js'; // → import './lib/utils.js' (relative to chunk)

// Original: absolute specifier → kept absolute in output
import '/project/lib/utils.js'; // → import '/project/lib/utils.js'
```

The idea: if you wrote a relative import, you probably want a relative import in the output. If you wrote an absolute import, you probably meant it to stay absolute.

##### `true`

Always convert absolute external paths to relative:

```js
// Both become relative in output
import './lib/utils.js'; // → import './lib/utils.js'
import '/project/lib/utils.js'; // → import '../lib/utils.js'
```

When converting an absolute path to a relative path, Rolldown does _not_ take the [`file`](/reference/OutputOptions.file) or [`dir`](/reference/OutputOptions.dir) options into account, because those may not be present e.g. for builds using the JavaScript API. Instead, it assumes that the root of the generated bundle is located at the common shared parent directory of all modules that were included in the bundle. If the output chunk is itself nested in a subdirectory by choosing e.g. `chunkFileNames: "chunks/[name].js"`, the relative path is adjusted accordingly.

##### `false`

Never convert. All paths are kept as-is. Relative specifiers are **not** normalized internally either, which means two files importing `'./utils'` from different directories may be treated as the same external module.

```js
import './lib/utils.js'; // → import './lib/utils.js' (as-is)
import '/project/lib/utils.js'; // → import '/project/lib/utils.js' (as-is)
```

::: warning Deduplication issue with `false`
Setting `makeAbsoluteExternalsRelative: false` disables the normalization of relative specifiers. This means `'./utils'` imported from `src/a.js` and `'./utils'` imported from `src/b/c.js` may be treated as the same external module, even though they refer to different files. Use `false` only if you are certain all your external specifiers are already unique (e.g. bare package names).
:::

#### Example

Given `import '/project/lib/utils.js'` (absolute specifier) in an external module, with output at `dist/index.js`:

| `makeAbsoluteExternalsRelative` | Output path               |
| ------------------------------- | ------------------------- |
| `true`                          | `'../lib/utils.js'`       |
| `"ifRelativeSource"` (default)  | `'/project/lib/utils.js'` |
| `false`                         | `'/project/lib/utils.js'` |

Given `import './lib/utils.js'` (relative specifier):

| `makeAbsoluteExternalsRelative` | Output path        |
| ------------------------------- | ------------------ |
| `true`                          | `'./lib/utils.js'` |
| `"ifRelativeSource"` (default)  | `'./lib/utils.js'` |
| `false`                         | `'./lib/utils.js'` |

The three settings only produce different results for **absolute specifiers**.
