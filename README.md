This extension exposes the abstract syntax tree generated by PHP 7.

Installation
------------

**On Windows**: Download a [prebuilt Windows DLL](http://windows.php.net/downloads/pecl/snaps/ast/)
and move it into the `ext/` directory of your PHP instalation. Furthermore add
`extension=php_ast.dll` to your `php.ini` file.

**On Unix**: Compile and install the extension as follows.

```sh
phpize
./configure
make
sudo make install
```

Additionally add `extension=ast.so` to your `php.ini` file.

API overview
------------

Defines:

 * `ast\Node` class
 * `ast\Node\Decl` class
 * `ast\AST_*` kind constants (mirroring `zend_ast.h`)
 * `ast\flags\*` flags
 * `ast\parse_file(string $filename, int $version)`
 * `ast\parse_code(string $code, int $version [, string $filename = "string code"])`
 * `ast\get_kind_name(int $kind)`
 * `ast\kind_uses_flags(int $kind)`

Usage
-----

Code can be parsed using either `ast\parse_code()`, which accepts a code string, or
`ast\parse_file()`, which accepts a file path. Additionally both functions require a `$version`
argument to ensure forward-compatibility. The current version is 40.

```php
$ast = ast\parse_code('<?php ...', $version=40);
// or
$ast = ast\parse_file('file.php', $version=40);
```

The abstract syntax tree returned by these functions consists of `ast\Node` objects.
`ast\Node` is declared as follows:

```php
namespace ast;
class Node {
    public $kind;
    public $flags;
    public $lineno;
    public $children;
}
```

The `kind` property specified the type of the node. It is an integral value, which corresponds to
one of the `ast\AST_*` constants, for example `ast\AST_STMT_LIST`. You can retrieve the string name
of an integral kind by passing it to `ast\get_kind_name()`.

The `flags` property contains node specific flags. It is always defined, but for most nodes it is
always zero. `ast\kind_uses_flags()` can be used to determine whether a certain kind has a
meaningful flags value. Which nodes use which flags is explained in the "Flags" section below.

The `lineno` property specifies the *starting* line number of the node.

The `children` property contains an array of child-nodes. These children can be either other
`ast\Node` objects or plain values. The meaning of the children is node-specific and should be
deduced from context or by looking at the [parser definition][parser].

Function and class declarations use `ast\Node\Decl` objects instead, which specify a number of
additional properties:

```php
namespace ast\Node;
use ast\Node;

class Decl extends Node {
    public $endLineno;
    public $name;
    public $docComment;
}
```

`endLineno` provides the end line number of the declaration, `name` contains the name of the
function or class (can be `null` for anonymous classes) and `docComment` contains the preceding
doc comment or `null` if no doc comment was used.

Simple usage example:

```php
<?php

$code = <<<'EOC'
<?php
$var = 42;
EOC;

var_dump(ast\parse_code($code, $version=40));

// Output:
object(ast\Node)#1 (4) {
  ["kind"]=>
  int(133)
  ["flags"]=>
  int(0)
  ["lineno"]=>
  int(1)
  ["children"]=>
  array(1) {
    [0]=>
    object(ast\Node)#2 (4) {
      ["kind"]=>
      int(517)
      ["flags"]=>
      int(0)
      ["lineno"]=>
      int(2)
      ["children"]=>
      array(2) {
        ["var"]=>
        object(ast\Node)#3 (4) {
          ["kind"]=>
          int(256)
          ["flags"]=>
          int(0)
          ["lineno"]=>
          int(2)
          ["children"]=>
          array(1) {
            ["name"]=>
            string(3) "var"
          }
        }
        ["expr"]=>
        int(42)
      }
    }
  }
}
```

The [`util.php`][util] file defines an `ast_dump()` function, which can be used to create a more
compact and human-readable dump of the AST structure:

```php
<?php

require 'path/to/util.php';

$code = <<<'EOC'
<?php
$var = 42;
EOC;

echo ast_dump(ast\parse_code($code, $version=40)), "\n";

// Output:
AST_STMT_LIST
    0: AST_ASSIGN
        var: AST_VAR
            name: "var"
        expr: 42
```

To additionally show line numbers pass the `AST_DUMP_LINENOS` option as the second argument to
`ast_dump()`.

A more substantial AST dump can be found [in the tests][test_dump].

Flags
-----

This section lists which flags are used by which AST node kinds. The "combinable" flags can be
combined using bitwise or and should be checked by using `$ast->flags & ast\flags\FOO`. The
"exclusive" flags are used standalone and should be checked using `$ast->flags === ast\flags\BAR`.

```
// Used by ast\AST_ARRAY_ELEM and ast\AST_CLOSURE_VAR (exclusive)
1 = by-reference

// Used by ast\AST_NAME (exclusive)
ast\flags\NAME_FQ (= 0)
ast\flags\NAME_NOT_FQ
ast\flags\NAME_RELATIVE

// Used by ast\AST_METHOD, ast\AST_PROP_DECL, ast\AST_CLASS_CONST_DECL
// and ast\AST_TRAIT_ALIAS (combinable)
ast\flags\MODIFIER_PUBLIC
ast\flags\MODIFIER_PROTECTED
ast\flags\MODIFIER_PRIVATE
ast\flags\MODIFIER_STATIC
ast\flags\MODIFIER_ABSTRACT
ast\flags\MODIFIER_FINAL

// Used by ast\AST_CLOSURE (combinable)
ast\flags\MODIFIER_STATIC

// Used by ast\AST_FUNC_DECL, ast\AST_METHOD, ast\AST_CLOSURE (combinable)
ast\flags\FUNC_RETURNS_REF  // legacy alias: ast\flags\RETURNS_REF
ast\flags\FUNC_GENERATOR    // used only in PHP >= 7.1

// Used by ast\AST_CLOSURE_VAR
ast\flags\CLOSURE_USE_REF

// Used by ast\AST_CLASS (exclusive)
ast\flags\CLASS_ABSTRACT
ast\flags\CLASS_FINAL
ast\flags\CLASS_TRAIT
ast\flags\CLASS_INTERFACE
ast\flags\CLASS_ANONYMOUS

// Used by ast\AST_PARAM (exclusive)
ast\flags\PARAM_REF
ast\flags\PARAM_VARIADIC

// Used by ast\AST_TYPE (exclusive)
ast\flags\TYPE_ARRAY
ast\flags\TYPE_CALLABLE
ast\flags\TYPE_VOID         // since version 40
ast\flags\TYPE_BOOL         // since version 40
ast\flags\TYPE_LONG         // since version 40
ast\flags\TYPE_DOUBLE       // since version 40
ast\flags\TYPE_STRING       // since version 40
ast\flags\TYPE_ITERABLE     // since version 40
ast\flags\TYPE_OBJECT       // since version 45

// Used by ast\AST_CAST (exclusive)
ast\flags\TYPE_NULL
ast\flags\TYPE_BOOL
ast\flags\TYPE_LONG
ast\flags\TYPE_DOUBLE
ast\flags\TYPE_STRING
ast\flags\TYPE_ARRAY
ast\flags\TYPE_OBJECT

// Used by ast\AST_UNARY_OP (exclusive)
ast\flags\UNARY_BOOL_NOT
ast\flags\UNARY_BITWISE_NOT
ast\flags\UNARY_MINUS       // since version 20
ast\flags\UNARY_PLUS        // since version 20
ast\flags\UNARY_SILENCE     // since version 20

// Used by ast\AST_BINARY_OP and ast\AST_ASSIGN_OP in version >= 20 (exclusive)
ast\flags\BINARY_BITWISE_OR
ast\flags\BINARY_BITWISE_AND
ast\flags\BINARY_BITWISE_XOR
ast\flags\BINARY_CONCAT
ast\flags\BINARY_ADD
ast\flags\BINARY_SUB
ast\flags\BINARY_MUL
ast\flags\BINARY_DIV
ast\flags\BINARY_MOD
ast\flags\BINARY_POW
ast\flags\BINARY_SHIFT_LEFT
ast\flags\BINARY_SHIFT_RIGHT

// Used by ast\AST_BINARY_OP (exclusive)
ast\flags\BINARY_BOOL_AND            // since version 20
ast\flags\BINARY_BOOL_OR             // since version 20
ast\flags\BINARY_BOOL_XOR
ast\flags\BINARY_IS_IDENTICAL
ast\flags\BINARY_IS_NOT_IDENTICAL
ast\flags\BINARY_IS_EQUAL
ast\flags\BINARY_IS_NOT_EQUAL
ast\flags\BINARY_IS_SMALLER
ast\flags\BINARY_IS_SMALLER_OR_EQUAL
ast\flags\BINARY_IS_GREATER          // since version 20
ast\flags\BINARY_IS_GREATER_OR_EQUAL // since version 20
ast\flags\BINARY_SPACESHIP
ast\flags\BINARY_COALESCE            // since version 40

// Used by ast\AST_ASSIGN_OP in versions before 20 (exclusive)
ast\flags\ASSIGN_BITWISE_OR
ast\flags\ASSIGN_BITWISE_AND
ast\flags\ASSIGN_BITWISE_XOR
ast\flags\ASSIGN_CONCAT
ast\flags\ASSIGN_ADD
ast\flags\ASSIGN_SUB
ast\flags\ASSIGN_MUL
ast\flags\ASSIGN_DIV
ast\flags\ASSIGN_MOD
ast\flags\ASSIGN_POW
ast\flags\ASSIGN_SHIFT_LEFT
ast\flags\ASSIGN_SHIFT_RIGHT

// Used by ast\AST_MAGIC_CONST (exclusive)
ast\flags\MAGIC_LINE
ast\flags\MAGIC_FILE
ast\flags\MAGIC_DIR
ast\flags\MAGIC_NAMESPACE
ast\flags\MAGIC_FUNCTION
ast\flags\MAGIC_METHOD
ast\flags\MAGIC_CLASS
ast\flags\MAGIC_TRAIT

// Used by ast\AST_USE, ast\AST_GROUP_USE and ast\AST_USE_ELEM (exclusive)
ast\flags\USE_NORMAL
ast\flags\USE_FUNCTION
ast\flags\USE_CONST

// Used by ast\AST_INCLUDE_OR_EVAL (exclusive)
ast\flags\EXEC_EVAL
ast\flags\EXEC_INCLUDE
ast\flags\EXEC_INCLUDE_ONCE
ast\flags\EXEC_REQUIRE
ast\flags\EXEC_REQUIRE_ONCE

// Used by ast\AST_ARRAY (exclusive), since PHP 7.1
ast\flags\ARRAY_SYNTAX_SHORT
ast\flags\ARRAY_SYNTAX_LONG
ast\flags\ARRAY_SYNTAX_LIST
```

AST node kinds
--------------

This section lists the AST node kinds that are supported and the names of their child nodes (in
version >= 30).

```
AST_AND:              left, right            // prior to version 20
AST_ARRAY_ELEM:       value, key
AST_ASSIGN:           var, expr
AST_ASSIGN_OP:        var, expr
AST_ASSIGN_REF:       var, expr
AST_BINARY_OP:        left, right
AST_BREAK:            depth
AST_CALL:             expr, args
AST_CAST:             expr
AST_CATCH:            class, var, stmts
AST_CLASS:            extends, implements, stmts
                      name, docComment       // since version 50
AST_CLASS_CONST:      class, const
AST_CLONE:            expr
AST_CLOSURE:          params, uses, stmts, returnType
                      name, docComment       // since version 50
AST_CLOSURE_VAR:      name
AST_COALESCE:         left, right            // prior to version 40
AST_CONDITIONAL:      cond, true, false
AST_CONST:            name
AST_CONST_ELEM:       name, value
AST_CONTINUE:         depth
AST_DECLARE:          declares, stmts
AST_DIM:              expr, dim
AST_DO_WHILE:         stmts, cond
AST_ECHO:             expr
AST_EMPTY:            expr
AST_EXIT:             expr
AST_FOR:              init, cond, loop, stmts
AST_FOREACH:          expr, value, key, stmts
AST_FUNC_DECL:        params, uses, stmts, returnType
                      name, docComment       // since version 50
AST_GLOBAL:           var
AST_GOTO:             label
AST_GREATER:          left, right            // prior to version 20
AST_GREATER_EQUAL:    left, right            // prior to version 20
AST_GROUP_USE:        prefix, uses
AST_HALT_COMPILER:    offset
AST_IF_ELEM:          cond, stmts
AST_INCLUDE_OR_EVAL:  expr
AST_INSTANCEOF:       expr, class
AST_ISSET:            var
AST_LABEL:            name
AST_MAGIC_CONST:
AST_METHOD:           params, uses, stmts, returnType
                      name, docComment       // since version 50
AST_METHOD_CALL:      expr, method, args
AST_METHOD_REFERENCE: class, method
AST_NAME:             name
AST_NAMESPACE:        name, stmts
AST_NEW:              class, args
AST_NULLABLE_TYPE:    type                   // Used only in PHP 7.1
AST_OR:               left, right            // Prior to version 20
AST_PARAM:            type, name, default
AST_POST_DEC:         var
AST_POST_INC:         var
AST_PRE_DEC:          var
AST_PRE_INC:          var
AST_PRINT:            expr
AST_PROP:             expr, prop
AST_PROP_ELEM:        name, default
AST_REF:              var
AST_RETURN:           expr
AST_SHELL_EXEC:       expr
AST_SILENCE:          expr                   // prior to version 20
AST_STATIC:           var, default
AST_STATIC_CALL:      class, method, args
AST_STATIC_PROP:      class, prop
AST_SWITCH:           cond, stmts
AST_SWITCH_CASE:      cond, stmts
AST_THROW:            expr
AST_TRAIT_ALIAS:      method, alias
AST_TRAIT_PRECEDENCE: method, insteadof
AST_TRY:              try, catches, finally
AST_TYPE:
AST_UNARY_MINUS:      expr                   // prior to version 20
AST_UNARY_OP:         expr
AST_UNARY_PLUS:       expr                   // prior to version 20
AST_UNPACK:           expr
AST_UNSET:            var
AST_USE_ELEM:         name, alias
AST_USE_TRAIT:        traits, adaptations
AST_VAR:              name
AST_WHILE:            cond, stmts
AST_YIELD:            value, key
AST_YIELD_FROM:       expr

// List nodes (numerically indexed children):
ZEND_AST_ARG_LIST
ZEND_AST_ARRAY
ZEND_AST_CATCH_LIST
ZEND_AST_CLASS_CONST_DECL
ZEND_AST_CLOSURE_USES
ZEND_AST_CONST_DECL
ZEND_AST_ENCAPS_LIST
ZEND_AST_EXPR_LIST
ZEND_AST_IF
ZEND_AST_LIST
ZEND_AST_NAME_LIST
ZEND_AST_PARAM_LIST
ZEND_AST_PROP_DECL
ZEND_AST_STMT_LIST
ZEND_AST_SWITCH_LIST
ZEND_AST_TRAIT_ADAPTATIONS
ZEND_AST_USE
```

Version changelog
-----------------

### 50 (in development)

* `ast\Node\Decl` nodes are no longer generated. AST kinds `AST_FUNCTION`, `AST_METHOD`,
  `AST_CLOSURE` and `AST_CLASS` now also use the normal `ast\Node` class. The `name` and
  `docComment` properties are now represented as children. The `endLineno` is still represented as
  an (undeclared) property.
* An integer `__declId` has been added to declaration nodes of kind `AST_FUNCTION`, `AST_METHOD`,
  `AST_CLOSURE` and `AST_CLASS`. The `__declId` uniquely identifies a declaration within the parsed
  code and will remain the same if the code is parsed again. This is useful to distinguish closures
  declared on the same line, or multiple conditional declarations using the same name. The ID is not
  unique across different codes/files.
* `\ast\parse_file` will now consistently return an empty statement list (similar to
  `\ast\parse_code`) if it is was passed a zero-byte file. Previously, it would return `null`.

### 45 (in development)

This version normalizes the AST to PHP 7.2 format.

* An `object` type annotation now returns an `AST_TYPE` with `TYPE_OBJECT` flag, rather than
  treating `object` as a class name.

### 40 (current)

Supported since 2017-01-18.

* `AST_COALESCE` is now represented as an `AST_BINARY_OP` with flag `BINARY_COALESCE`.
* For `AST_NAME` nodes with `NAME_FQ` the leading backslash is now dropped if syntax like
  `('\bar')()` is used. Previously this would return the name as `'\bar'`, while a normal `\bar()`
  call would return it as `'bar'`. Now always the latter form is used.
* `null` elements are now stripped from `AST_STMT_LIST`. Previously these could be caused by nop
  statements (`;`).
* Type hints `int`, `float`, `string`, `bool`, `void` and `iterable` will now be represented as
  `AST_TYPE` nodes with a respective flag.
* Many `stmts` children could previously hold one of `null`, a single node or an `AST_STMT_LIST`.
  These will now be normalized to always use an `AST_STMT_LIST`. A `null` is only allowed if it is
  semantically meaningful, e.g. in the case of `declare(ticks=1);` vs `declare(ticks=1) {}`.

### 35 (supported)

Supported since 2016-08-04.

This version normalizes the AST to PHP 7.1 format.

* The `class` node of `AST_CATCH` is now always represented as an `AST_NAME_LIST`. In lower
  versions: On PHP 7.0 it will always be an `AST_NAME`. In PHP 7.1 it will be an `AST_NAME` if
  there is only a single class and `AST_NAME_LIST` otherwise.
* `list()` destructuring is now always represented as an `AST_ARRAY` with `ARRAY_SYNTAX_LIST` flag.
  In lower versions: On PHP 7.0 it will always be an `AST_LIST`. In PHP 7.1 it will be an
  `AST_ARRAY` if `list()` is used with keys. In PHP 7.1 destructuring using `[]` will always be
  represented using `AST_ARRAY`, independently of the version.

### 30 (deprecated)

Supported since 2015-03-10. Deprecated since 2017-06-25.

* Use string names for child nodes of kinds with fixed length.

### 20 (removed)

Supported since 2015-12-14. Deprecated since 2016-08-04. Removed since 2017-01-18.

* `AST_GREATER`, `AST_GREATER_EQUAL`, `AST_OR`, `AST_AND` nodes are now represented using
  `AST_BINARY_OP` with flags `BINARY_IS_GREATER`, `BINARY_IS_GREATER_OR_EQUAL`, `BINARY_BOOL_OR`
  and `BINARY_BOOL_AND`.
* `AST_SILENCE`, `AST_UNARY_MINUS` and `AST_UNARY_PLUS` nodes are now represented using
  `AST_UNARY_OP` with flags `UNARY_SILENCE`, `UNARY_MINUS` and `UNARY_PLUS`
* `AST_ASSIGN_OP` now uses `BINARY_*` flags instead of separate `ASSIGN_*` flags.
* `AST_STATIC` and `AST_CATCH` now use an `AST_VAR` node for the static/catch variable. Previously
  it was a simple string containing the name.
* Nested `AST_STMT_LIST`s are now flattened out.

### 15 (removed)

Supported since 2015-10-21. Deprecated since 2016-04-30. Removed since 2016-08-04.

* In line with an upstream change, the `docComment` property on `AST_PROP_DECL` has been moved to
  `AST_PROP_ELEM`. This means that each property in one property declaration can have its own
  doc comment.

### 10 (removed)

Initial. Removed since 2016-08-04.

Differences to PHP-Parser
-------------------------

Next to php-ast I also maintain the [PHP-Parser][php-parser] library, which has some overlap with
this extension. This section summarizes the main differences between php-ast and PHP-Parser so you
may decide which is preferable for your use-case.

The primary difference is that php-ast is a PHP extension (written in C) which exports the AST
internally used by PHP 7. PHP-Parser on the other hand is library written in PHP. This has a number
of consequences:

 * php-ast is significantly faster than PHP-Parser, because the AST construction is implemented in
   C.
 * php-ast needs to be installed as an extension, on Linux either by compiling it manually or
   retrieving it from a package manager, on Windows by loading a DLL. PHP-Parser is installed as a
   Composer dependency.
 * php-ast only runs on PHP >= 7.0, as prior versions did not use an internal AST. PHP-Parser
   supports PHP >= 5.4.
 * php-ast may only parse code that is syntactically valid on the version of PHP it runs on. This
   means that it's not possible to parse code using features of newer versions (e.g. PHP 7.1 code
   while running on PHP 7.0). Similarly, it is not possible to parse code that is no longer
   syntactically valid on the used version (e.g. some PHP 5 code may no longer be parsed -- however
   most code will work). PHP-Parser supports parsing both newer and older (up to PHP 5.2) versions.
 * php-ast only provides the starting line number (and for declarations the ending line number) of
   nodes, because this is the only part that PHP itself stores. PHP-Parser provides precise file
   offsets.

There are a number of differences in the AST representation and available support code:

 * The PHP-Parser library uses a separate class for every node type, with child nodes stored as
   type-annotated properties. php-ast uses one class for everything, with children stored as
   arrays. The former approach is friendlier to developers because it has very good IDE integration.
   The php-ast extension does not use separate classes, because registering hundreds of classes was
   judged a no-go for a bundled extension.
 * The PHP-Parser library contains various support code for working with the AST, while php-ast only
   handles AST construction. The reason for this is that implementing this support code in C is
   extremely complicated and there is little other benefit to implementing it in C. The main
   components that PHP-Parser offers that may be of interest are:
    * Node dumper (human readable representation): While the php-ast extension does not directly
      implement this, a `ast_dump` function is provided in the `util.php` file.
    * Pretty printer (converting the AST back to PHP code): This is not provided natively, but the
      [php-ast-reverter][php-ast-reverter] package implements this functionality.
    * Name resolution (resolving namespace prefixes and aliases): There is currently no standalone
      package for this.
    * AST traversation / visitation: There is currently no standalone package for this either, but
      implementing a recursive AST walk is easy.

  [parser]: http://lxr.php.net/xref/PHP_TRUNK/Zend/zend_language_parser.y
  [util]: https://github.com/nikic/php-ast/blob/master/util.php
  [test_dump]: https://github.com/nikic/php-ast/blob/master/tests/001.phpt
  [php-parser]: https://github.com/nikic/PHP-Parser
  [php-ast-reverter]: https://github.com/tpunt/php-ast-reverter
