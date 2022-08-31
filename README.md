# erlte - ERLang TEmplate library

Fast Erlang template library with bundling support for HTML, JavaScript and CSS files.

[![Erlang CI](https://github.com/ergenius/erlte/actions/workflows/erlang.yml/badge.svg)](https://github.com/ergenius/erlte/actions/workflows/erlang.yml)

## Motivation
The need for erlte arose while using good old sgte in various web applications. I was quite happy with the simplicity of sgte for many years, but concerned about the rendering speed. When I finally replaced sgte with my new shiny library, I could see around 5x times faster rendering speed!

## Features
- Fast rendering.
- Simple. Reading this file should be enough for understanding most use case scenarios.
- Supports replacing blocks of delimited code with your own values both at compilation or rendering time (variables).
- Supports escaping HTML variables. I wouldn't call that 'secure', but it's better than nothing.
- Supports bundling HTML files using custom HTML elements.
- Supports bundling JavaScript files using native JavaScript side effect static import declarations.
- Supports bundling CSS files using CSS @import rules for strings.
- Supports writing and reading compiled templates.
- Templates default variables delimiters are friendly with most HTML, CSS or JavaScript editors and browsers.
- Can also be used with strings or binaries instead of files.

## Limitations
- Only JavaScript static import declarations are supported.
- CSS @import rules work fine only with file strings (no list of media queries or url are supported).
- erlte "compiler" is far away from a full HTML/JS/CSS parser. It behaves well for most scenarios, but some corner cases may exist. As long as you use single line imports and you don't nest unescaped comments or imports in strings, you should be fine.
- erlte does not support any conditionals or pseudocode and will probably never do. For some, this is a limitation, for me, it's a feature. The price of implementing this will be a much slower rendering speed. I also don't like polluting my HTML templates with spaghetti code. We have JavaScript for that.

## Alternatives
- sgte - a simple Erlang Template Engine https://github.com/filippo/sgte 

## Documentation

- Github pages: https://ergenius.github.io/erlte/
- '/doc' subdirectory (generated by ex_doc)


## Recommended flow
- Compile your template. Avoid compiling the template for each rendering!
- Cache the compilation result for later usage. Helpers for saving compiled templates to files are provided.
- Render the compiled template.

## Examples

### Erlang examples

Compile and render a file template:

```erlang
{ok, Compiled} = erlte:compile({file, "/path/to/template/file.html"}),
Variables = [{language, "Erlang"}, {designed_by, "Joe Armstrong, Robert Virding and Mike Williams"}],
{ok, Rendered} = erlte:render(Compiled, Variables).
```

Compile and render a list:

```erlang
{ok, Compiled} = erlte:compile("{{project}} was designed by {{author}}"),
Variables = [{project, "erlte"}, {author, "Madalin"}],
{ok, Rendered} = erlte:render(Compiled, Variables).
```

Compile a binary:

```erlang
{ok, Compiled} = erlte:compile(<<"{{project}} was designed by {{author}}">>),
```

You are free to mix ANY atoms, lists, binaries or integer numbers for variable names:

```erlang
{ok, Compiled} = erlte:compile("{{molecule}} {{server is down}} {{1}}"),
Variables = [{molecule, "atoms are OK"}, {<<"server is down">>, "binaries are OK"}, {1, "integers are OK"}],
{ok, Rendered} = erlte:render(Compiled, Variables).
```

You can also specify a function that returns the value to be rendered instead of the variable value.
The function must be exported from the specified module and must have arity = 4. 
The function will receive the template file format, the variable name, variable value, argument as parameters and must return a binary.
Argument is atom undefined in the example below but can be any Erlang term().

```erlang
% Define the function somewhere in a module (remember to export the function).
% If anything goes wrong with your function including if your function does not return a binary, an exception will be trown!
-export([your_function/4]).
your_function(TemplateFormat, VariableName, VariableValue, Arg) -> <<"do_some_magic_here">>.
% Later you can use this function to render any variable you want 
Variables = [{variable_name, {VariableValue, {f, your_module_name, your_function, undefined}}}].
```

Save the compiled template to a file ('erlte' extension is recommended):

```erlang
{ok, Compiled} = erlte:compile("{{language}} was designed by {{designed_by}}"),
ok = erlte:compiled_write_file("/path/to/template/compiled.erlte", Compiled).
```

Read a compiled template from a file and render it:

    > {ok, Compiled} = erlte:compiled_read_file("/path/to/template/compiled.erlte").    
    > {ok, Rendered} = erlte:render(Compiled).

### Templates examples

Let's have a quick look at some examples. 
You can find more templates samples into `examples/templates` project directory.

HTML demonstrating erlte-import custom HTML element

```html
<erlte-import>./fragments/html-begin.html</erlte-import>
<erlte-import>./fragments/head.html</erlte-import>
<body>
<!-- <erlte-import>./test/invalid/commented/import</erlte-import> -->
<!-- Test single line comment -->
<!-- Test multiline 
     comment -->
<h1>Hello world from {{erlte}}!</h1>
<!-- Test comment with {{variable}} --> 
<erlte-import>./fragments/footer.html</erlte-import>
</body>
<erlte-import>./fragments/html-end.html</erlte-import>
```

JavaScript import

```js
import "./config"; // You can import without .js file extension
import "./erlte";
import "./utils/base.js"; // You can import with file extension
import "./utils/validator";

// Let's call some functions
erlte.S.Utils.Base.erlang();
erlte.S.Utils.Validator.isString("Erlang");
```

CSS import

```css
@import "test1.css";
/* a { color: blue } */
/**/
/* */
div {
     /* inside */
     color: green;
     /* between */
     border-radius: 3px / 7px
     /* end */
}
```

## Under the hood

#### How is compiling implemented?
- When ertle compiles the template, it produces a record containing one atom (holding original template file format) and one list (holding binaries and variables tuples).
- erlte also supports variables being used at the compilation stage. Good candidates are variables that rarely change. Not having to locate or replace them for each render may dramatically improve your rendering speed.

Example of template and variables at compilation time:

```erlang
Template = <<"{{constant}} {{anonfun}}! Mixing variables populated at compilation time with {{data}} at rendering time!">>,
CompilationVariables = [
        {constant, "Hello"},
        %% Using your own function for rendering the variable is also possible
        {anonfun, {"world", {f, module_name, function_name, undefined}}}
],
%% You can see erlte is smart enogh to merge the proper fragments toghether at compilation phase
%% skipping only remaining variables!
{ok, #erlte_compiled{
   format = txt,
   fragments = [
    <<"Hello world! Mixing variables populated at compilation time with ">>,
    {v, <<"data">>},
    <<" at rendering time!">>
    ]
}} = erlte:compile({file, "/path/to/template/file.html"}, CompilationVariables).
```

#### How is rendering implemented?
- ertle iterates your rendering variables once and replaces the variables in the compiled template list with the specified values.
- If the template contains variable names that are not specified at the rendering phase, the original variable names are kept untouched.
- All variable names that are not found in the template are simply ignored.
- At the end, you get an iolist of binaries. That's all folks!

## Project roadmap

1. Continuously fixing bugs and tuning performance.
2. Writing more testing units.
3. Keeping it simple and fast.

## Erlang versions supported

erlte officially supports OTP release 20 and later.

Development takes place using OTP 25 release and tests are done on:
- 25.0.3
- 24.3.4
- 23.3.4
- 22.3.4
- 21.3.8
- 20.3.8

Unofficially, you may be able to use erlte with older Erlang versions. No guarantee included.

## Dependencies

None at this moment, but I intend to introduce erlhtml in the next version. erlhtml is Erlang library for URL encoding and escaping HTML entities mantained by the same author as erlte.

## Authors

- Madalin Grigore-Enescu (ergenius) <github@ergenius.com>

## License

erlte is available under the MIT license (see `LICENSE`).

