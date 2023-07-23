# safe-handlebars

A fork of the `@kbn/handlebars` custom version of the handlebars package which, to improve security, does not use `eval` or `new Function`. This means that templates can't be compiled into JavaScript functions in advance and hence, rendering the templates is a lot slower.

## Attribution and package future

This package was created to make consumption of `eval-less` handlebars simple to do. If the Kibana project intends to upload their own fork as a standalone package, or get it integrated into handlebars core, then this project will become deprecated in deference to that.

Big thanks to the Kibana contributors for implementing this!

## Limitations

- Only the following compile options are supported:

  - `data`
  - `knownHelpers`
  - `knownHelpersOnly`
  - `noEscape`
  - `strict`
  - `assumeObjects`
  - `preventIndent`
  - `explicitPartialContext`

- Only the following runtime options are supported:
  - `data`
  - `helpers`
  - `partials`
  - `decorators` (not documented in the official Handlebars [runtime options documentation](https://handlebarsjs.com/api-reference/runtime-options.html))
  - `blockParams` (not documented in the official Handlebars [runtime options documentation](https://handlebarsjs.com/api-reference/runtime-options.html))

## Implementation differences

The standard `handlebars` implementation:

1. When given a template string, e.g. `Hello {{x}}`, return a "render" function which takes an "input" object, e.g. `{ x: 'World' }`.
1. The first time the "render" function is called the following happens:
   1. Turn the template string into an Abstract Syntax Tree (AST).
   1. Convert the AST into a hyper optimized JavaScript function which takes the input object as an argument.
   1. Call the generate JavaScript function with the given "input" object to produce and return the final output string (`Hello World`).
1. Subsequent calls to the "render" function will re-use the already generated JavaScript function.

The custom `@kbn/handlebars` implementation:

1. When given a template string, e.g. `Hello {{x}}`, return a "render" function which takes an "input" object, e.g. `{ x: 'World' }`.
1. The first time the "render" function is called the following happens:
   1. Turn the template string into an Abstract Syntax Tree (AST).
   1. Process the AST with the given "input" object to produce and return the final output string (`Hello World`).
1. Subsequent calls to the "render" function will re-use the already generated AST.

_Note: Not parsing of the template string until the first call to the "render" function is deliberate as it mimics the original `handlebars` implementation. This means that any errors that occur due to an invalid template string will not be thrown until the first call to the "render" function._

## Technical details

The `handlebars` library exposes the API for both [generating the AST](https://github.com/handlebars-lang/handlebars.js/blob/master/docs/compiler-api.md#ast) and walking it by implementing the [Visitor API](https://github.com/handlebars-lang/handlebars.js/blob/master/docs/compiler-api.md#ast-visitor). We can leverage that to our advantage and create our own "render" function, which internally calls this API to generate the AST and then the API to walk the AST.

The `@kbn/handlebars` implementation of the `Visitor` class implements all the necessary methods called by the parent `Visitor` code when instructed to walk the AST. They all start with an upppercase letter, e.g. `MustacheStatement` or `SubExpression`. We call this class `ElasticHandlebarsVisitor`.

To parse the template string to an AST representation, we call `Handlebars.parse(templateString)`, which returns an AST object.

The AST object contains a bunch of nodes, one for each element of the template string, all arranged in a tree-like structure. The root of the AST object is a node of type `Program`. This is a special node, which we do not need to worry about, but each of its direct children has a type named like the method which will be called when the walking algorithm reaches that node, e.g. `ContentStatement` or `BlockStatement`. These are the methods that our `Visitor` implementation implements.

To instruct our `ElasticHandlebarsVisitor` class to start walking the AST object, we call the `accept()` method inherited from the parent `Visitor` class with the main AST object. The `Visitor` will walk each node in turn that is directly attached to the root `Program` node. For each node it traverses, it will call the matching method in our `ElasticHandlebarsVisitor` class.

To instruct the `Visitor` code to traverse any child nodes of a given node, our implementation needs to manually call `accept(childNode)`, `acceptArray(arrayOfChildNodes)`, `acceptKey(node, childKeyName)`, or `acceptRequired(node, childKeyName)` from within any of the "node" methods, otherwise the child nodes are ignored.

### State

We keep state internally in the `ElasticHandlebarsVisitor` object using the following private properties:

- `contexts`: An array (stack) of `context` objects. In a simple template this array will always only contain a single element: The main `context` object. In more complicated scenarios, new `context` objects will be pushed and popped to and from the `contexts` stack as needed.
- `output`: An array containing the "rendered" output of each node (normally just one element per node). In the most simple template, this is simply joined together into a the final output string after the AST has been traversed. In more complicated templates, we use this array temporarily to collect parameters to give to helper functions (see the `getParams` function).

## Debugging

### Print AST

To output the generated AST object structure in a somewhat readable form, use the following script:

```sh
./packages/kbn-handlebars/scripts/print_ast.js
```

Example:

```sh
./packages/kbn-handlebars/scripts/print_ast.js '{{value}}'
```

Output:

```js
{
  type: 'Program',
  body: [
    {
      type: 'MustacheStatement',
      path: {
        type: 'PathExpression',
        data: false,
        depth: 0,
        parts: [ 'value' ],
        original: 'value'
      },
      params: [],
      hash: undefined,
      escaped: true
    }
  ]
}
```

By default certain properties will be hidden in the output.
For more control over the output, check out the options by running the script without any arguments.

### Print generated code

It's possible to see the generated JavaScript code that `handlebars` create for a given template using the following command line tool:

```sh
./node_modules/handlebars/print-script <template> [options]
```

Options:

- `-v`: Enable verbose mode.

Example:

```sh
./node_modules/handlebars/print-script '{{value}}' -v
```

You can pretty print just the generated code using this command:

```sh
./node_modules/handlebars/print-script '{{value}}' | \
  node -e 'process.stdin.on(`data`, c => console.log(`(${eval(`(${c})`).code})`))' | \
  npx prettier --write --stdin-filepath template.js | \
  npx cli-highlight -l javascript
```
