# Conditional Module Features

## Status

Champion(s): TBD

Stage: 0

## Motivation

JavaScript modules today are all-or-nothing. When a consumer imports from a module, the entire module and all its dependencies are loaded and executed, even if the consumer only needs a subset of the module's functionality.

Bundlers partially address this through tree-shaking, but tree-shaking operates on bindings, not on class methods, prototype augmentation, or other runtime structures. It also relies on heuristics that vary across tools and cannot work in unbundled environments like native browser ESM.

The "deferred re-exports" proposal (`export defer`) attempted to address this, but its optimization is silently defeated by namespace imports (`import *`), creating a fragile contract where the exporter opts in but the consumer can unknowingly opt out.

## Proposal

This proposal introduces **module features**: a mechanism for modules to declare optional chunks of functionality, and for consumers to declare which features they need.

### Using features

Features are applied at import time using the new `with { features: ['feature-name'] }` import attribute:

```js
import { x } from './mod.js' with { features: ['name'] }
import * as ns from './mod.js' with { features: ['name'] }
await import('./mod.js', { with: { features: ['name'] } })
```

This import attribute is **non-keying** in the sense that the module has a single instance in the module registry, regardless of the features attribute value. It is effectively removed entirely from the canonical attributes list used in keying.

The default import mode without features specified is for all features to be disabled by default.

The enabled feature set for a module is the union of all features requested by its static importers. This set is frozen before any module executes.

```js
// a.js
import { App } from './ui.js' with { features: ['ssr'] }

// b.js
import { App } from './ui.js' with { features: ['routing'] }
```

Both a.js and b.js receive the same App. The module ui.js is evaluated with both ssr and routing enabled, because the resolved set is the union.

Once the module is loaded it cannot be loaded with new features later on as it has already been loaded.

### Feature guards

A new feature guard static syntax is supported - via a `when <name>` **strawman syntax for now**. The features provided by a given module are the union of these static guard names.

Guards apply to static imports, static exports and blocks.

### Example

```js
// ui.js

import { render } from './core.js'

// conditional import binding that only applies when a feature is enabled.
// binding is not even defined when the feature guard is disabled
import { hydrate } from './ssr.js' when ssr
import { Router } from './router.js' when routing

export { render }

// export binding is omitted when the feature is disabled
export { hydrate } when ssr

export class App {
  render() { /* ... */ }

  hydrate() {
    return hydrate(this);
  } when ssr

  navigate(path) {
    return Router.push(path);
  } when routing
}
```

### Feature reexports

Feature reexports can be used to replicate deferred reexports style functionality:

```js
export { foo } from './foo.js' when foo
```

### Auto features

Features guarding export statements can be considered "auto features" which automatically enable whenever importing that name explicitly.

This way, the `features` attribute can be optional for those features.

Example:

module.js
```js
export { bar } from './bar.js' when featureA

export function foo () {

} when featureB


{
    console.log('featureA enabled');
} when featureA

{
    console.log('featureB enabled');
} when featureA
```

Usage:

```js
import { foo } from './module.js';
```

By importing `foo`, it is clear that the `featureB` feature must be enabled by the declaration, therefore it is auto-enabled.

As a result, `'featureA enabled'` is logged when executing the module.

This way, it is possible to add features to existing libraries supporting feature guards based on import choice.

### Semantics

* Disabled imports: The binding exists but the module is not fetched. The binding value is undefined.
* Disabled exports: The export name is present in the module's export list. Its value is undefined.
* Disabled class elements: The method or field is not defined on the prototype or instance.
* Disabled when blocks: The block is skipped during evaluation.
* Feature names are module-scoped. feature x on module A and feature x on module B are independent.
* Feature sets are static. They are resolved from the union of static importers before evaluation and cannot change.

## FAQ

### How is this different from export defer?
export defer places the optimization on the export side and relies on consumers using named imports. Namespace imports (import *) silently defeat it. This proposal makes the contract explicit on both sides: the exporter declares what's optional, the consumer declares what's needed.

### How is this different from tree-shaking?
Tree-shaking operates at the binding level and requires whole-program analysis by a bundler. This proposal works at runtime, handles class methods and prototype augmentation, and provides well-defined semantics rather than tool-specific heuristics.

### Why not runtime if checks?
Runtime conditionals can't prevent module loading. when is static — the host can skip fetching, parsing, and linking entire subgraphs for disabled features.

### Why union semantics?
Modules are singletons in ESM. If two consumers import the same module with different features, the module can only be evaluated once. The union ensures all consumers get what they requested, at the cost of potentially loading more than any single consumer needs.