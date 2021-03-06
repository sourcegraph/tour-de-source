# How TypeScript ESLint Works, a source-level introduction

![TypeScript and ESLint logos](https://user-images.githubusercontent.com/1646931/169179393-3f1db84a-080a-4887-9f9f-78ab9dc29d4c.png)

[TypeScript ESLint](https://typescript-eslint.io) is the de facto tool for linting in TypeScript.
It's used by many thousands of projects and ensures code quality, security, and best practices across the TypeScript world.
If you're deploying TypeScript into production, chances are you're using this in your toolchain.

In this post, we provide a hands-on, technical introduction to how this essential static analysis tool works by tracing through its key code paths.
We'll cover the following:

- How ESLint rules are defined
- How ESLint rules use ASTs to analyze code files
- Switching between ESLint's and TypeScript's AST formats
- Calling TypeScript's type checking APIs to analyze code

> Not familiar with ASTs yet?
> No worries!
> Read [TypeScript ESLint's Abstract Syntax Trees (ASTs) docs page](https://typescript-eslint.io/docs/development/architecture/asts) to learn how programs represent and reason about source code.

Read on!

## Overview

TypeScript ESLint is implemented as a collection of packages that build on [ESLint](https://eslint.org).
The packages we'll be going over in this walkthrough are:

- [`eslint-plugin`](https://www.npmjs.com/package/@typescript-eslint/eslint-plugin), which defines the ESLint plugin and the linting rules.
- [`parser`](https://www.npmjs.com/package/@typescript-eslint/parser) and [`typescript-estree`](https://www.npmjs.com/package/@typescript-eslint/typescript-estree), which are responsible for parsing TypeScript source code into an ESLint-compatible form, mapping from TypeScript ASTs to ESLint's AST representation (ESTree) and providing an API to query the AST from the rules.

### ESLint Rules

The purpose of a linter is to detect a set of patterns in source code that match common anti-patterns.
ESLint, like many linters, defines each anti-pattern to search for in **rules**.
ESLint's docs contain a [list of core ESLint rules](https://eslint.org/docs/rules) available for all JavaScript projects.
TypeScript ESlint provides an additional [list of rules specific to TypeScript projects](https://typescript-eslint.io/rules).

For example, the `no-extra-non-null-assertion` rule catches unnecessary repeated `!`s in TypeScript code.
Here is an example of that anti-pattern:

```ts
const scream =
  Math.random() > 0.5 ? "What's your favorite scary movie?" : undefined;

// Multiple !!s are used, but only one is needed
console.log(scream!!!.toUpperCase());
```

Let's dig into how we can create a rule to ban those unnecessary repeated `!`s.

## Declaring a Rule

ESLint rules are declared as objects that have two properties:

- `meta`: Rule metadata such as configuration options and messages it may report
- `create`: A function that creates the functions to be called by ESLint at runtime

We're going to dive into the definition of the `no-extra-non-null-assertion` rule in source.
It is short enough to include here in its entirety:

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint@3eab889022c9d1617f275017d6951f663ea57f24/-/blob/packages/eslint-plugin/src/rules/no-extra-non-null-assertion.ts

Note that rules declared in the TypeScript ESLint project use a `util.createRule` helper that internally calls to an `ESLintUtils.RuleCreator`:

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint@3eab889022c9d1617f275017d6951f663ea57f24/-/blob/packages/eslint-plugin/src/util/createRule.ts

`RuleCreator` adds a documentation URL to rules under `meta.docs.url` based on their name.
TypeScript ESLint rules are each documented on a page under https://typescript-eslint.io/rules, such as https://typescript-eslint.io/rules/no-extra-non-null-assertion.

### Rule Metadata Properties

Two particularly useful rule metadata properties are specified in many TypeScript ESLint rules:

- `fixable`: If the rule includes an auto-fixer on any of its messages, either `"code"` or `"whitespace"`
- `messages`: An object keying ID strings to full messages that may be reported by the rule

`no-extra-non-null-assertion`'s `meta` indicates it has an auto-fixer for code issues and that it may report _`Forbidden extra non-null assertion`_ if it detects an issue.

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint@3eab889022c9d1617f275017d6951f663ea57f24/-/blob/packages/eslint-plugin/src/rules/no-extra-non-null-assertion.ts?L6-17

### Rule Creator Functions

A rule's `create` function returns an object whose keys are AST queries and whose values are functions that will be run whenever a node matching the corresponding query is found.
The AST query keys use the [ESQuery](https://github.com/estools/esquery) library, which allows using syntax similar to CSS selectors for querying types of nodes.

`no-extra-non-null-assertion` creates three node queries in its returned object:

1. Any `!` non-null assertion within another `!` non-null assertion: e.g. `scream!!.length`
2. Any `!` non-null assertion just before an `?.` optional chained property: e.g. `scream!?.length`
3. Any `!` non-null assertion just before an `?.` optional chained call: e.g. `scream!?.()`

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint@3eab889022c9d1617f275017d6951f663ea57f24/-/blob/packages/eslint-plugin/src/rules/no-extra-non-null-assertion.ts?L19-39

Note that any AST node names prefixed with `TS`, including `TSNonNullExpression`, are TypeScript-specific nodes.
These are available to ESLint rules when using [`@typescript-eslint/parser`](http://npmjs.com/package/@typescript-eslint/parser) instead of ESLint's built-in parser.
The `@typescript-eslint/utils` package exports those types under the `TSESTree` namespace.

The corresponding TypeScript types for all the possible node types are stored in a [`@typescript-eslint/types`](https://npmjs.com/package/@typescript-eslint/types) package.
All nodes extend from a `BaseNode` interface:

https://sourcegraph.com/npm/typescript-eslint/types@3eab889022c9d1617f275017d6951f663ea57f24/-/blob/dist/generated/ast-spec.d.ts?L236-L244

#### Reporting on Code Issues

Rule creator functions receive an ESLint "Context" object as a `context` parameter.
That `context` object is most commonly used for its `report` method, which is what indicates a code issue has been found.

`no-extra-non-null-assertion` calls `context.report` immediately for any node that matches one of its three node queries.
It provides three properties:

- `node`: The node itself, indicating the text range of the report (where to put unhappy squiggly underlines in an IDE)
- `messageId`: Which string ID of message to report, from `meta`
- `fix`: A function that generate instructions for how to auto-fix the code issue

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint@3eab889022c9d1617f275017d6951f663ea57f24/-/blob/packages/eslint-plugin/src/rules/no-extra-non-null-assertion.ts?L20-30

## Type-checking

One powerful distinction between TypeScript ESLint and vanilla ESLint is that TypeScript ESLint rules have access to the TypeScript type checker.
Rules that use type checking are able to make informed decisions on nodes based on their type information.

TypeScript ESLint's `no-for-in-array` rule uses type information to determine if a `for-in` loop is iterating over an array rather than an object or string.
Knowing the type of a value is only possible in JavaScript or TypeScript code if you have access to a type checker.
Here is an example of this pattern:

https://sourcegraph.com/github.com/cypress-io/cypress@3eab889022c9d1617f275017d6951f663ea57f24/-/blob/packages/driver/src/cypress/proxy-logging.ts?L13

This anti-pattern is quite widespread, because it feels intuitive to use the for-in syntax to traverse an array.
Doing so, however, may visit the array elements out-of-order.
Instead `for-of` loops or `array.forEach` are recommended.

ESLint rules that use type information are set up similarly to other ESLint rules.
Before we jump into how they use the type checker, the source of `no-for-in-array` is also small enough to be seen in its entirety here:

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint@3eab889022c9d1617f275017d6951f663ea57f24/-/blob/packages/eslint-plugin/src/rules/no-for-in-array.ts

## Getting a Type Checker

The `no-for-in-array` rule enables a `meta.docs.requiresTypeChecking` property indicating it plans on using type checking powered by a TypeScript type checker.

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint@3eab889022c9d1617f275017d6951f663ea57f24/-/blob/packages/eslint-plugin/src/rules/no-for-in-array.ts?L10

A TypeScript type checker may be retrieved by a rule during its runtime by getting TypeScript ESLint's "parser services" for its `getTypeChecker()` API:

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint@3eab889022c9d1617f275017d6951f663ea57f24/-/blob/packages/eslint-plugin/src/rules/no-for-in-array.ts?L23-24

### Dual ASTs

One complication from using TypeScript in ESLint rules is that TypeScript type checkers don't work with the same AST format as ESLint nodes.
TypeScript uses its own AST format.

- _TSESTree_ (ESLint and TypeScript ESLint): The representation used by ESLint internally, and by TypeScript ESLint to enable the plugin to reuse much of the code and linting framework provided by ESLint
- _TypeScript_: The representation emitted by the TypeScript compiler and what the type checker's APIs expect as inputs

TypeScript ESLint's parser services include an `esTreeNodeToTSNodeMap` mapping that allows retrieving the _TypeScript_ from the _TSESTree_ node:

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint@3eab889022c9d1617f275017d6951f663ea57f24/-/blob/packages/eslint-plugin/src/rules/no-for-in-array.ts?L25

Now that the rule has obtained the _TSESTree_ node, the node can be passed to TypeScript's type checker's APIs.

### Type Checking APIs

Many TypeScript ESLint rules use a `getConstrainedTypeAtLocation` utility that combines of two type checking APIs to determine the type of a node:

1. `getTypeAtLocation`: Given a _TypeScript_ node, its TypeScript type (`ts.Type`)
2. `getBaseConstraintOfType`: If the TypeScript type is a generic from a type parameter, any constraint that limits what it's allowed to be (e.g. a `string[]` from `function myFunction<T extends string[]>`)

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint@3cd54b7c12a07c113160ef694636721ea66b4f69@3eab889022c9d1617f275017d6951f663ea57f24/-/blob/packages/eslint-plugin/src/rules/no-for-in-array.ts?L27:30#tab=def

The rule then checks if the type is in some way an array, union of arrays, or a string:

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint@3cd54b7c12a07c113160ef694636721ea66b4f69@3eab889022c9d1617f275017d6951f663ea57f24/-/blob/packages/eslint-plugin/src/rules/no-for-in-array.ts?L33-L34#tab=def

If the type was one of those disallowed types, then the rule reports an issue with the original _TSESTree_ node.

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint@HEAD@3eab889022c9d1617f275017d6951f663ea57f24/-/blob/packages/eslint-plugin/src/rules/no-for-in-array.ts?L36-39

## Next steps

TypeScript ESLint is an amazing tool that brings the power of static analysis to multitudes of TypeScript projects.
It is also an impressive case study of code reuse.
Rather than build an entire linting project from scratch, it finds an elegant way to build on top of JavaScript's ESLint while integrating the TypeScript compiler to enable rules to take advantage of type information.

There are many ways to get involved:

- [See what sorts of rules and autofixes TypeScript ESLint can apply in the playground](https://typescript-eslint.io/play)
- [Use TypeScript ESLint for your TypeScript codebase](https://typescript-eslint.io/docs/linting)
- [Create a new custom linting rule](https://typescript-eslint.io/docs/development/custom-rules)
- [Support this open-source project financially](https://opencollective.com/typescript-eslint/contribute)
