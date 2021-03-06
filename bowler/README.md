Bowler
======


Overview
--------

Bowler is a refactoring tool for manipulating Python at the syntax tree level. It
enables safe, large scale code modification while guaranteeing that the resulting code
compiles and runs. It provides both a simple command line interface and a fluent API in
Python for generating complex code modifications in code.

    query = (
        Query([<file paths>])
        # rename class Foo to Bar
        .select_class("Foo")
        .rename("Bar")
        # change method buzz(x) to buzzard(x: int)
        .select_method("buzz")
        .rename("buzzard")
        .modify_argument("x", type_annotation="int")
    )

    query.diff()  # generate unified diff on stdout
    query.write()  # write changes directly to files

Bowler uses the concrete syntax tree (CST) as generated by the [lib2to3][] module.


CLI Reference
-------------

Using Bowler at the command line follows the pattern below:

    $ bowler [--debug] <command> [--help] [<arguments> ...]

Bowler supports the following commands:

    do [<query> ...]
        Compile and run the given query, or open an IPython shell if none given.
        Common API elements will already be available in the global namespace.

    dump [<path>  ...]
        Dump the CST from the given paths to stdout.

    rename_function [-i | --interactive] <old_name> <new_name> [<path>  ...]
        Rename a function and its calls.


Query Reference
---------------

Queries use a fluent API to build a series of transforms over a given set of paths.
Each transform consists of a selector, any number of filters, and one or more
modifiers.  Queries will only be compiled and executed once an appropriate action
is triggered – like `diff()` or `write()`.

Constructing queries should follow this basic pattern:

1. Create the query object, and specify all paths that should be considered
2. Specify a selector to define broad search criteria
3. Optionally specify one or more filters to refine the scope of modification
4. Specify one or more modifiers
5. Repeat from step 2 to include more transforms in the query
6. Execute the query with a terminal action, such as `diff()` or `write()`.

Queries are started by creating a `Query` instance, and passing a list of paths that
should be considered for modification:

    query = Query([path, ...])

All methods on a `Query` object will return the same `Query` object back, enabling
"fluent" usage of the API – chaining one method call after the other:

    query = Query(...).selector(...)[.filter(...)].modifier(...)


### Selectors

Selectors are query methods that generate a search pattern for the custom [lib2to3][]
syntax.  There are a number of prewritten selectors, but Bowler supports arbitrary
selectors as well.

Bowler supports the following methods for choosing selectors:

    .select_root()
        Selects the root of the syntax tree for each file.

    .select_module(name)
        Selects all module imports and references with the given name.

    .select_class(name)
        Selects all class definitions for – or subclasses of – the given name, as well
        as any calls or references to that name.

    .select_subclass(name)
        Selects all class definitions that subclass the given name, as well as any calls
        or references to that name.

    .select_attribute(name)
        Selects all class or object attributes, including assignments and references.

    .select_method(name)
        Selects all class method definitions with the given name, as well as any method
        calls or references with that name.

    .select_function(name)
        Selects all bare function definitions with the given name, as well as any calls
        or references with that name.

    .select_var(name)
        Select all references to that name, regardless of context.

    .select_pattern(pattern)
        Select nodes based on the arbitrary [lib2to3][] pattern given.


### Filters

Filters are functions that limit the scope of modifiers.  They are functions with the
signature of `filter(node, capture, filename) -> bool`, and return `True` if the current
node should be eligible for modification, or `False` to skip the node.

- `node` refers to the base CST node matched by the active selector
- `capture` is a dictionary, mapping named captures to their associated CST leaf or node
- `filename` is the current filename being modified

Bowler supports the following methods for adding filters:

    .is_call()
        Filters all nodes that aren't function or method calls.

    .is_def()
        Filters all nodes that aren't function or method definitions.

    .in_class(name, [include_subclasses = True])
        Filters all nodes that aren't part of the either given class definition or
        a subclass of the given class.

    .is_filename([include = <regex>], [exclude = <regex>])
        Filters all nodes belonging to files that don't match the given include/exclude
        regular expressions.

    .add_filter(function | str)
        Use an arbitrary function to filter nodes. If given a string, compile that
        and `eval()` it at each node to determine if the filter passed.


### Modifiers

Modifiers take each matched node – that passed all active filters – and optionally
applies some number of modifications to the CST of that node. They are functions with
the signature of `filter(node, capture, filename)`, with no expected return value.

- `node` refers to the base CST node matched by the active selector
- `capture` is a dictionary, mapping named captures to their associated CST leaf or node
- `filename` is the current filename being modified

Bowler supports the following methods for adding modifiers:

    .rename(new_name)
        Rename all `*_name` captures to the given new name.

    .encapsulate([internal_name])
        Encapsulate a class attribute into an `@property` decorated getter and setter.
        Requires the `select_attribute()` selector.

    .add_argument(name, value, [positional], [after], [type_annotation])
        Add a new argument to a method or function, with a default value and optional
        position or type annotation. Also updates all callers with the new argument.

    .modify_argument(name, [new_name], [type_annotation], [default_value])
        Modify an existing argument to a method or function, optionally renaming it,
        adding/changing the type annotation, or adding/changing the default value.
        Also updates all callers with new names.

    .remove_argument(name)
        Remove an existing argument from a method or function, as well as from callers.

    .add_modifier(function | str)
        Add an arbitrary modifier function. If given a string, compile that and
        `exec()` it at each matched node to perform modifications.


### Actions

After building one or more transforms, those transforms are applied through the use of
terminal actions.  These include generating diffs, writing modifications to disk, and
dumping matched nodes to stdout.

Bowler supports the following terminal actions:

    .diff([interactive=False])
        Generate a unified diff and echo it to stdout. Colors will be used when stdout
        is a terminal. Interactive mode will prompt the user after each hunk, with
        actions similar to those found in `git add -p`.

    .idiff()
        Shortcut for `.diff(interactive=True)`.

    .write()
        Write all changes back to disk, overwriting existing files.

    .dump()
        For each node matched, for each transform, print the CST representation of the
        node along with all captured nodes.

    .execute([write=False], [interactive=False])
        Longer form of `.diff()` or `.write()`.


[lib2to3]: https://docs.python.org/3.6/library/2to3.html#module-lib2to3
