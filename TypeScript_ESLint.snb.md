# How TypeScript ESLint Works, a source-level introduction

TypeScript ESLint is the de facto standard for linting in TypeScript. It's used by hundreds of thousands of projects and ensures code quality, security, and best practices across the TypeScript world. If you're deploying TypeScript into production, chances are you're using this in your toolchain.

In this post, we provide a hands-on, technical introduction to how this essential static analysis tool works by tracing through its key code paths.

## Overview

TypeScript ESLint is implemented as an ESLint plugin. It has a few main modules, the most important of which are:

* `eslint-plugin`, which defines the ESLint plugin and the linting rules.
* `parser` and `typescript-estree`, which are responsible for parsing TypeScript source code into an ESLint-compatible form, mapping from TypeScript ASTs to ESLint's AST representation (ESTree) and providing an API to query the AST from the rules.

## How rules work

The purpose of the linter is to detect a set of patterns in source code that match common anti-patterns. These patterns are described in **rules**. To dive into the code, let's start with the definition of one such rule.

The `no-for-in-array` rule disallows iterating over an array with a for-in loop. Here is an example of this pattern:

https://sourcegraph.com/github.com/cypress-io/cypress/-/blob/packages/driver/src/cypress/proxy-logging.ts?L13

This anti-pattern is quite widespread, because it feels intuitive to use the for-in syntax to traverse an array. Doing so, however, may visit the array elements out-of-order; instead `array.forEach` is recommended.

As an entrypoint into the code, let's dive into the definition of this rule:

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint/-/blob/packages/eslint-plugin/src/rules/no-for-in-array.ts


After the metadata, the rule logic is defined here:

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint/-/blob/packages/eslint-plugin/src/rules/no-for-in-array.ts?L22-41

There may be some unfamiliar syntax for some here. This code might look like it's defining a function called `ForInStatement` within an object declaration, but it's really just a terser variant of the following (declaring a function value for the key "ForInStatement" in the returned object):

```typescript
{
    ForInStatement: (node): void => { ... }
}
```

This key will be used to match this rule with the name of an AST node type in the ESLint AST, so that the node the function takes as argument will always be the node type that the rule targets:

https://sourcegraph.com/npm/typescript-eslint/types/-/blob/dist/generated/ast-spec.d.ts?L75

The rule will make use of the TypeScript type checker:

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint/-/blob/packages/eslint-plugin/src/rules/no-for-in-array.ts?L23-24

It also deals with **two ASTs**: (1) the ESTree representation used by ESLint and (2) the TypeScript AST emitted by the TypeScript compiler.

The `node` value passed as an argument to the rule function is from AST #1, an instance of [`TSESTree.Node`](https://sourcegraph.com/npm/typescript-eslint/types/-/blob/dist/generated/ast-spec.d.ts?L641):

The `originalNode` value is from AST #2, an instance of ([`TSNode`](https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint@3eab889022c9d1617f275017d6951f663ea57f24/-/blob/packages/typescript-estree/src/ts-estree/ts-nodes.ts?L18:13#tab=references)):

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint@HEAD/-/blob/packages/eslint-plugin/src/rules/no-for-in-array.ts?L25#tab=def

The TypeScript AST has type information, which is often used in linting rules. This rule checks if the type of the variable in the expression is an array type:

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint@HEAD/-/blob/packages/eslint-plugin/src/rules/no-for-in-array.ts?L32-35#tab=def

## Dueling ASTs

TypeScript ESLint maintains two ASTs, because each provides a distinct set of useful functionality.

1. The `TSESTree` representation is used by ESLint internally and enables the plugin to reuse much of the code and linting framework provided by ESLint.
2. The `TSNode` representation is emitted by the TypeScript compiler and contains type information.

In order to use both in conjunction, the plugin must create both and construct a mapping between the two. The way it does so is by using a co-recursive switch pattern quite common in AST constructors and traversers. Let's examine how it does this.

The key function is `parseAndGenerateServices`:

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint@HEAD/-/blob/packages/typescript-estree/src/parser.ts?L499-636

This function compiles the TypeScript from source:

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint@3eab889022c9d1617f275017d6951f663ea57f24/-/blob/packages/typescript-estree/src/parser.ts?L595-602&subtree=true

...and then converts the TypeScript AST to an ESTree-compatible one:

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint@3eab889022c9d1617f275017d6951f663ea57f24/-/blob/packages/typescript-estree/src/parser.ts?L605-611&subtree=true

The `astConverter` function makes use of the `Converter` class:

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint@3eab889022c9d1617f275017d6951f663ea57f24/-/blob/packages/typescript-estree/src/ast-converter.ts?L26-29&subtree=true

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint@3eab889022c9d1617f275017d6951f663ea57f24/-/blob/packages/typescript-estree/src/convert.ts?L66-85&subtree=true#tab=def

Within the `Converter` class, the `convertProgram` method kicks off the AST conversion:

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint@3eab889022c9d1617f275017d6951f663ea57f24/-/blob/packages/typescript-estree/src/convert.ts?L93-95&subtree=true

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint@3eab889022c9d1617f275017d6951f663ea57f24/-/blob/packages/typescript-estree/src/convert.ts?L105-110&subtree=true

The `converter` method it invokes is co-recursive with `convertNode`, which is just a giant switch statement on the AST node type that constructs the appropriate ESLint node corresponding to the current TypeScript node. The two corresponding nodes are then added to the TypeScript-to-ESLint node mapping:

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint@3eab889022c9d1617f275017d6951f663ea57f24/-/blob/packages/typescript-estree/src/convert.ts?L127-132

https://sourcegraph.com/github.com/typescript-eslint/typescript-eslint@3eab889022c9d1617f275017d6951f663ea57f24/-/blob/packages/typescript-estree/src/convert.ts?L770-836&subtree=true

In this fashion, the TypeScript AST is converted by recursively traversing the AST and building the new ESLint one along the way. Both ASTs and the mapping between them are stored so they can be referenced by rules.

## Conclusion

TypeScript ESLint is an amazing tool that brings the power of static analysis to multitudes of TypeScript projects. It is also an impressive case study of code reuse. Rather than build an entire linting project from scratch, it finds an elegant way to build on top of JavaScript's ESLint while integrating the TypeScript compiler to enable rules to take advantage of type information.

There are many ways to get involved:

* [See what sorts of rules and autofixes it can apply in the playground](https://typescript-eslint.io/play/)
* [Use it for your TypeScript codebase](https://typescript-eslint.io/docs/linting/)
* [Contribute a new linting rule](https://typescript-eslint.io/docs/development/custom-rules)
* [Support this open-source project financially](https://opencollective.com/typescript-eslint/contribute)
