# ECMAScriptParser

**Champions**: Finding one...

**Author**: Jack Works

**Stage**: N/A

This proposal describes adding an ECMAScriptParser to JavaScript. Just like [DOMParser](http://mdn.io/DOMParser) in HTML and [Houdini's parser API in CSS](https://github.com/WICG/CSS-Parser-API/blob/master/README.md).

## The problem and opportunity

### Usage A: Check if a piece of code is syntax-correct

Some "feature detection" (See: https://github.com/Tokimon/es-feature-detection/blob/master/syntax/es2019.json) use `eval` to check if some syntax-level feature is available. Like

```js
try {
    eval('async function x() {}')
    asyncFunctionSupported = true
} catch {}
```

Or in some other scenarios, we just want to check if a piece of code is syntax-correct instead of `eval` it.

## Usage B: Sandbox

Some sandbox proposal/libraries do need an AST transformer to transform or reject the dangerous JS code.

A sandbox proposal, [realms](https://github.com/tc39/proposal-realms/), it's [polyfill](https://github.com/Agoric/realms-shim) is using RegExp to [reject "HTML style comments"](https://github.com/Agoric/realms-shim/blob/025d975da12c8033022c271e5d99a3810066dfa4/src/sourceParser.js#L19) and [reject ESModules](https://github.com/Agoric/realms-shim/blob/025d975da12c8033022c271e5d99a3810066dfa4/src/sourceParser.js#L55). Using RegExp is very likely to make false positives on valid codes.

But if they want to reduce false positives, they have to load a babel, typescript or whatever what parser into the library and run it to transform the code into the safe pattern.

### Due to the lack of built-in parser, they imported ...

-   Babel as a parser in realms-shim (https://github.com/Agoric/realms-shim/pull/15)
-   TypeScript Compiler as a parser in (https://github.com/Agoric/realms-shim/issues/47)
-   TypeScript as a transformer to transform ESModule into SystemJS (https://github.com/Jack-Works/loader-with-esmodule-example/)

### False positive rejection examples:

-   [#34: rejectSomeDirectEvalExpressions is too aggressive](https://github.com/Agoric/realms-shim/issues/34)
-   [#37: rejectHtmlComments throws on bignumber.js comment](https://github.com/Agoric/realms-shim/issues/37)
-   [#39: Any code with import expression inside string literal or comment throws error](https://github.com/Agoric/realms-shim/issues/39)
-   ... and so on

## Usage C: Join the parser step

This part may be hard to accomplish. Need to discuss if it is needed or even possible.

Just like CSS Houdini's parser can parse CSS in a programmable way, this ECMAScript parser may let developer handle the step of parsing the source code in a programmable way.

Don't know what the API should be like, just for example:

```js
const JSXElement = new ECMAScriptParser.Grammar('JSXElement', [
    JSXSelfClosingElement,
    new ECMAScriptParser.Grammar(
        JSXOpeningElement,
        {
            type: JSXChildren,
            optional: true
        },
        JSXClosingElement
    )
])
const jsxParser = new ECMAScriptParser()
const primaryExpression = parser.getGrammar('PrimaryExpression')
primaryExpression.extends(JSXElement)
jsxParser.parse(`const expr = <a />`)
```

# Goal

-   Generate AST from source code
-   Generate source code from an AST
-   [Maybe] a built-in AST walker to replace AST Nodes on demand
-   (If AST is not plain object,) provide a way to construct new AST Nodes.
-   [Maybe] a set of API that extends ECMAScript Parser (like we can create a special parser for [JSX](https://facebook.github.io/jsx/)).

# API design

```ts
class ECMAScriptParser {
    parse(source: string): ECMAScriptAST

    compile(ast: ECMAScriptAST): string

    static visitChildNodes(mapFunction: (beforeTransform: ECMAScriptAST) => ECMAScriptAST): ECMAScriptAST

    // [Maybe]
    addGrammar(gr: Grammar): this
    getGrammar(grName: string): Grammar | null
    replaceGrammar(gr: Grammar): this
    deleteGrammar(gr: Grammar): this
    removeGrammar(gr: Grammar): this
    static Grammar = class Grammar {
        // [wait for discussion]
    }
}
```

## What shape is type ECMAScriptAST?

wait for discussion

see also:

-   https://github.com/tc39/proposal-binary-ast
-   https://github.com/rricard/proposal-const-value-types

## Example usage

### Used to check if new Syntax is supported

```js
try {
    new ECMAScriptParser().parse(`
const res = await fetch(jsonService)
case (res) {
  when {status: 200, headers: {'Content-Length': s}} ->
    console.log(\`size is \${s}\`),
  when {status: 404} ->
    console.log('JSON not found'),
  when {status} if (status >= 400) -> {
    throw new RequestError(res)
  },
}`)
} catch {
    // https://github.com/tc39/proposal-pattern-matching
    console.log('Your browser does not support Pattern Matching')
}
```

### Use in Realms API

```js
const parser = new ECMAScriptParser()
const secure = new Realms({
    transforms: [
        {
            rewrite: context => {
                const ast = parser.parse(context.src)
                ECMAScriptParser.visitChildNodes(node => {
                    if (node.kind === ECMAScriptParser.SyntaxKind.WithStatement) {
                        throw new SyntaxError('with statement is not supported')
                    } else if (node.kind === ECMAScriptParser.SyntaxKind.ImportDeclaration) {
                        return ECMAScriptParser.createImportDeclaration(
                            node.decorators,
                            node.modifiers,
                            node.importClause,
                            ECMAScriptParser.createStringLiteral(
                                '/safe-import-transformer.js?src=' + node.moduleSpecifier.toString()
                            )
                        )
                    }
                    return node
                })
                return context
            }
        }
    ]
})

secure.evaluate(`with (window) {}`)
// SyntaxError: with statement is not supported
source.evaluate(`import x from './y.js'`)
// will evaluate: `import x from '/safe-import-transformer.js?src=./y.js'`
```

### [Maybe] Using in enhance ECMAScript's syntax

```js
import { enhanceParser, jsxCompiler } from 'react-experimental-jsx-parser-in-browser'
/*
JSX extends the PrimaryExpression in the ECMAScript 6th Edition (ECMA-262) grammar:
PrimaryExpression :
    JSXElement
    JSXFragment
*/
const jsxParser = enhanceParser(new ECMAScriptParser())
const ast = jsxParser.parse(`const link = <a />`)
const code = jsxParser.compile(
    jsxCompiler({
        jsxFactory: 'React'
    })
)
console.log(code)
// const link = React.createElement('a', {})
```
