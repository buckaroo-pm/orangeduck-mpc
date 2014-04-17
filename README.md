Micro Parser Combinators
========================

Version 0.8.2


About
-----

_mpc_ is a lightweight and powerful Parser Combinator library for C.

Using _mpc_ might be of interest to you if you are...

* Building a new programming language
* Building a new data format
* Parsing an existing programming language
* Parsing an existing data format
* Embedding a Domain Specific Language
* Implementing [Greenspun's Tenth Rule](http://en.wikipedia.org/wiki/Greenspun%27s_tenth_rule)


Features
--------

* Type-Generic
* Predictive, Recursive Descent
* Easy to Integrate (One Source File in ANSI C)
* Automatic Error Message Generation
* Regular Expression Parser Generator
* Language/Grammar Parser Generator


Alternatives
------------

The current main alternative for a C based parser combinator library is a branch of [Cesium3](https://github.com/wbhart/Cesium3/tree/combinators).

_mpc_ provides a number of features that this project does not offer, and also overcomes a number of potential downsides:

* _mpc_ Works for Generic Types
* _mpc_ Doesn't rely on Boehm-Demers-Weiser Garbage Collection
* _mpc_ Doesn't use `setjmp` and `longjmp` for errors
* _mpc_ Doesn't pollute the namespace


Quickstart
==========

Here is how one would use _mpc_ to create a parser for a basic mathematical expression language.

```c
mpc_parser_t *Expr  = mpc_new("expression");
mpc_parser_t *Prod  = mpc_new("product");
mpc_parser_t *Value = mpc_new("value");
mpc_parser_t *Maths = mpc_new("maths");

mpca_lang(
  " expression : <product> (('+' | '-') <product>)*; "
  " product    : <value>   (('*' | '/') <value>  )*; "
  " value      : /[0-9]+/ | '(' <expression> ')';    "
  " maths      : /^/ <expression> /$/;               ",
  Expr, Prod, Value, Maths);

mpc_result_t r;

if (mpc_parse("input", input, Maths, &r)) {
  mpc_ast_print(r.output);
  mpc_ast_delete(r.output);
} else {
  mpc_err_print(r.error);
  mpc_err_delete(r.error);
}

mpc_cleanup(4, Expr, Prod, Value, Maths);
```

If you were to set `input` to the string `(4 * 2 * 11 + 2) - 5`, the printed output would look like this.

```
>
  regex
  expression|>
    value|>
      char:1:1 '('
      expression|>
        product|>
          value|regex:1:2 '4'
          char:1:4 '*'
          value|regex:1:6 '2'
          char:1:8 '*'
          value|regex:1:10 '11'
        char:1:13 '+'
        product|value|regex:1:15 '2'
      char:1:16 ')'
    char:1:18 '-'
    product|value|regex:1:20 '5'
  regex
```

Getting Started
===============

Introduction
------------

Parser Combinators are structures that encode how to parse a particular language. They can be combined using intuitive operators to create new parsers of increasing complexity. Using these operators detailed grammars and languages can be parsed and processed in a quick, efficient, and easy way.

The trick behind Parser Combinators is the observation that by structuring the library in a particular way, one can make building parser combinators look like writing a grammar itself. Therefore instead of describing _how to parse a language_, a user must only specify _the language itself_, and the computer will work out how to parse it ... as if by magic!

_mpc_ can be used in this mode, or, as shown in the above example, it can allow you to specify the grammar directly as a string or in a file.

Basic Parsers
-------------

### String Parsers

All the following functions construct new basic parsers of the type `mpc_parser_t *`. All of those parsers return a newly allocated `char *` with the character(s) they manage to match. If unsuccessful they will return an error. They have the following functionality.

* * * 

```c
mpc_parser_t *mpc_any(void);
```

Matches any individual character

* * * 

```c
mpc_parser_t *mpc_char(char c);
```

Matches a single given character `c`

* * *

```c
mpc_parser_t *mpc_range(char s, char e);
```

Matches any single given character in the range `s` to `e` (inclusive)

* * *

```c
mpc_parser_t *mpc_oneof(const char *s);
```

Matches any single given character in the string  `s`

* * *

```c
mpc_parser_t *mpc_noneof(const char *s);
```
Matches any single given character not in the string `s`

* * *

```c
mpc_parser_t *mpc_satisfy(int(*f)(char));
```

Matches any single given character satisfying function `f`

* * *

```c
mpc_parser_t *mpc_string(const char *s);
```

Matches exactly the string `s`


### Other Parsers

Several other functions exist that construct parsers with some other special functionality.

* * *

```c
mpc_parser_t *mpc_pass(void);
```

Consumes no input, always successful, returns `NULL`

* * *

```c
mpc_parser_t *mpc_fail(const char *m);
```

Consumes no input, always fails with message `m`.

* * *

```c
mpc_parser_t *mpc_failf(const char *fmt, ...);
```

Consumes no input, always fails with string formatted message given by `fmt` and following parameters.

* * *

```c
mpc_parser_t *mpc_lift(mpc_ctor_t f);
```

Consumes no input, always successful, returns the result of function `f`

* * *

```c
mpc_parser_t *mpc_lift_val(mpc_val_t *x);
```

Consumes no input, always successful, returns `x`

* * *

```c
mpc_parser_t *mpc_state(void);
```

Consumes no input, always successful, returns a copy of the parser state as `mpc_state_t *`. This pointer needs to be freed with `free` when done with.

* * *

```c
mpc_parser_t *mpc_anchor(int(*f)(char,char));
```

Consumes no input. Successful when function `f` returns true. Always returns `NULL` on success.

Function `f` is a _anchor_ function. It takes as input the last character parsed, and the next character in the input, and returns success or failure based upon these. This function can be set by the user to ensure some condition is met. This could be that the input is at a boundary between words and non-words, or anything else. The nice thing about this parser is that it consumes no input.

At the start of the input the first argument is set to `\0`. At the end of the input the second argument is set to the `\0`.



Parsing
-------

Once you've build a parser, you can run it on some input using one of the following functions. These functions return `1` on success and `0` on failure. They output either the result, or an error to a `mpc_result_t` variable. This type is defined as follows.

```c
typedef union {
  mpc_err_t *error;
  mpc_val_t *output;
} mpc_result_t;
```

where `mpc_val_t *` is synonymous with `void *` and simply represents some pointer to data - the exact type of which is dependant on the parser.  For almost all of the basic parsers the return type for a successful parser will be `char *`. 


* * *

```c
int mpc_parse(const char *filename, const char *string, mpc_parser_t *p, mpc_result_t *r);
```

Run a parser on some string.

* * *

```c
int mpc_parse_file(const char *filename, FILE *file, mpc_parser_t *p, mpc_result_t *r);
```

Run a parser on some file.

* * *

```c
int mpc_parse_pipe(const char *filename, FILE *pipe, mpc_parser_t *p, mpc_result_t *r);
```

Run a parser on some pipe (such as `stdin`).

* * *

```c
int mpc_parse_contents(const char *filename, mpc_parser_t *p, mpc_result_t *r);
```

Run a parser on the contents of some file.


Combinators
-----------

Combinators are functions that take one or more parsers and return a new parser of some given functionality. 

These combinators work independently of exactly what data type the parsers given as input return on success. In languages such as Haskell ensuring you don't input one type of data into a parser requiring a different type of data is done by the compiler. But in C we don't have that luxury. So it is at the discretion of the programmer to ensure that he or she deals correctly with the outputs of different parser types.

A second annoyance in C is that of manual memory management. Some parsers might get half-way and then fail. This means they need to clean up any partial data that has been collected in the parse. In Haskell this is handled by the Garbage Collector, but in C these combinators will need to take _destructor_ functions as input, which say how clean up any partial data that has been collected.

Here are the main combinators and how to use then.

* * *

```c
mpc_parser_t *mpc_expect(mpc_parser_t *a, const char *e);
mpc_parser_t *mpc_expectf(mpc_parser_t *a, const char *fmt, ...);
```

Returns a parser that runs `a`, and on success returns the result of `a`, while on failure reports that `e` was expected.

* * *

```c
mpc_parser_t *mpc_apply(mpc_parser_t *a, mpc_apply_t f);
mpc_parser_t *mpc_apply_to(mpc_parser_t *a, mpc_apply_to_t f, void *x);
```

Returns a parser that applies function `f` (optionality taking extra input `x`) to the result of parser `a`.

* * *

```c
mpc_parser_t *mpc_not(mpc_parser_t *a, mpc_dtor_t da);
mpc_parser_t *mpc_not_lift(mpc_parser_t *a, mpc_dtor_t da, mpc_ctor_t lf);
```

Returns a parser with the following behaviour. If parser `a` succeeds, then it fails and consumes no input. If parser `a` fails, then it succeeds, consumes no input and returns `NULL` (or the result of lift function `lf`). Destructor `da` is used to destroy the result of `a` on success.

* * *

```c
mpc_parser_t *mpc_maybe(mpc_parser_t *a);
mpc_parser_t *mpc_maybe_lift(mpc_parser_t *a, mpc_ctor_t lf);
```

Returns a parser that runs `a`. If `a` is successful then it returns the result of `a`. If `a` is unsuccessful then it succeeds, but returns `NULL` (or the result of `lf`).

* * *

```c
mpc_parser_t *mpc_many(mpc_fold_t f, mpc_parser_t *a);
```

Keeps running `a` until it fails. Results are combined using fold function `f`. See the _Function Types_ section for more details.

* * *

```c
mpc_parser_t *mpc_many1(mpc_fold_t f, mpc_parser_t *a);
```

Attempts to run `a` one or more times. Results are combined with fold function `f`.

* * *

```c
mpc_parser_t *mpc_count(int n, mpc_fold_t f, mpc_parser_t *a, mpc_dtor_t da);
```

Attempts to run `a` exactly `n` times. If this fails, any partial results are destructed with `da`. If successful results of `a` are combined using fold function `f`.

* * *

```c
mpc_parser_t *mpc_or(int n, ...);
```

Attempts to run `n` parsers in sequence, returning the first one that succeeds. If all fail, returns an error.

* * *

```c
mpc_parser_t *mpc_and(int n, mpc_fold_t f, ...);
```

Attempts to run `n` parsers in sequence, returning the fold of the results using fold function `f`. First parsers must be specified, followed by destructors for each parser, excluding the final parser. These are used in case of partial success. For example: `mpc_and(3, mpcf_strfold, mpc_char('a'), mpc_char('b'), mpc_char('c'), free, free);` would attempt to match `'a'` followed by `'b'` followed by `'c'`, and if successful would concatenate them using `mpcf_strfold`. Otherwise would use `free` on the partial results.

* * *

```c
mpc_parser_t *mpc_predictive(mpc_parser_t *a);
```

Returns a parser that runs `a` with backtracking disabled. This means if `a` consumes any input, it will not be reverted, even on failure. Turning backtracking off has good performance benefits for grammars which are `LL(1)`. These are grammars where the first character completely determines the parse result - such as the decision of parsing either a C identifier, number, or string literal. This option should not be used for non `LL(1)` grammars or it will produce incorrect results or crash the parser.

Another way to think of `mpc_predictive` is that it can be applied to a parser (for a performance improvement) if either successfully parsing the first character will result in a completely successful parse, or all of the referenced sub-parsers are also `LL(1)`.


Function Types
--------------

The combinator functions take a number of special function types as function pointers. Here is a short explanation of those types are how they are expected to behave. It is important that these behave correctly otherwise it is easy to introduce memory leaks or crashes into the system.

* * *

```c
typedef void(*mpc_dtor_t)(mpc_val_t*);
```

Given some pointer to a data value it will ensure the memory it points to is freed correctly.

* * *

```c
typedef mpc_val_t*(*mpc_ctor_t)(void);
```

Returns some data value when called. It can be used to create _empty_ versions of data types when certain combinators have no known default value to return. For example it may be used to return a newly allocated empty string.

* * *

```c
typedef mpc_val_t*(*mpc_apply_t)(mpc_val_t*);
typedef mpc_val_t*(*mpc_apply_to_t)(mpc_val_t*,void*);
```

This takes in some pointer to data and outputs some new or modified pointer to data, ensuring to free the input data if it is no longer used. The `apply_to` variation takes in an extra pointer to some data such as state of the system.

* * *

```c
typedef mpc_val_t*(*mpc_fold_t)(int,mpc_val_t**);
```

This takes a list of pointers to data values and must return some combined or folded version of these data values. It must ensure to free and input data that is no longer used once after combination has taken place.


Case Study - Identifier
=======================

Combinator Method
-----------------

Using the above combinators we can create a parser that matches a C identifier.

When using the combinators we need to supply a function that says how to combine two `char *`.

For this we build a fold function that will concatenate zero or more strings together. For this sake of this tutorial we will write it by hand, but this (as well as many other useful fold functions), are actually included in _mpc_ under the `mpcf_*` namespace, such as `mpcf_strfold`.

```c
mpc_val_t *strfold(int n, mpc_val_t **xs) {
  char *x = calloc(1, 1);
  int i;
  for (i = 0; i < n; i++) {
    x = realloc(x, strlen(x) + strlen(xs[i]) + 1);
    strcat(x, xs[i]);
    free(xs[i]);
  }
  return x;
}
```

We can use this to specify a C identifier, making use of some combinators to say how the basic parsers are combined.

```c
mpc_parser_t *alpha = mpc_or(2, mpc_range('a', 'z'), mpc_range('A', 'Z'));
mpc_parser_t *digit = mpc_range('0', '9');
mpc_parser_t *underscore = mpc_char('_');

mpc_parser_t *ident = mpc_and(2, strfold,
  mpc_or(2, alpha, underscore),
  mpc_many(strfold, mpc_or(3, alpha, digit, underscore)),
  free);

/* Do Some Parsing... */

mpc_delete(ident);
```

Notice that previous parsers are used as input to new parsers we construct from the combinators. Note that only the final parser `ident` must be deleted. When we input a parser into a combinator we should consider it to be part of the output of that combinator.

Because of this we shouldn't create a parser and input it into multiple places, or it will be doubly feed.


Regex Method
------------

There is an easier way to do this than the above method. _mpc_ comes with a handy regex function for constructing parsers using regex syntax. We can specify an identifier using a regex pattern as shown below.

```c
mpc_parser_t *ident = mpc_re("[a-zA-Z_][a-zA-Z_0-9]*");

/* Do Some Parsing... */

mpc_delete(ident);
```


Library Method
--------------

Although if we really wanted to create a parser for C identifiers, a function for creating this parser comes included in _mpc_ along with many other common parsers.

```c
mpc_parser_t *ident = mpc_ident();

/* Do Some Parsing... */

mpc_delete(ident);
```

Parser References
=================

Building parsers in the above way can have issues with self-reference or cyclic-reference. To overcome this we can separate the construction of parsers into two different steps. Construction and Definition.

* * *

```c
mpc_parser_t *mpc_new(const char *name);
```

This will construct a parser called `name` which can then be used as input to others, including itself, without fear of being deleted. Any parser created using `mpc_new` is said to be _retained_. This means it will behave differently to a normal parser when referenced. When deleting a parser that includes a _retained_ parser, the _retained_ parser will not be deleted along with it. To delete a retained parser `mpc_delete` must be used on it directly.

A _retained_ parser can then be defined using...

* * *

```c
mpc_parser_t *mpc_define(mpc_parser_t *p, mpc_parser_t *a);
```

This assigns the contents of parser `a` to `p`, and deletes `a`. With this technique parsers can now reference each other, as well as themselves, without trouble.

* * *

```c
mpc_parser_t *mpc_undefine(mpc_parser_t *p);
```

A final step is required. Parsers that reference each other must all be undefined before they are deleted. It is important to do any undefining before deletion. The reason for this is that to delete a parser it must look at each sub-parser that is used by it. If any of these have already been deleted a segfault is unavoidable - even if they were retained beforehand.

* * *

```c
void mpc_cleanup(int n, ...);
```

To ease the task of undefining and then deleting parsers `mpc_cleanup` can be used. It takes `n` parsers as input, and undefines them all, before deleting them all.

Note: _mpc_ may have separate stages for construction and definition, but it still does not detect [left-recursive grammars](http://en.wikipedia.org/wiki/Left_recursion). These will go into an infinite loop when they attempt to parse input, and so should specified instead in right-recursive form instead.


Library Reference
=================

Common Parsers
--------------

* `mpc_soi;` Matches only the start of input, returns `NULL`
* `mpc_eoi;` Matches only the end of input, returns `NULL`
* `mpc_boundary;` Matches only the boundary between words, returns `NULL`
* `mpc_whitespace` Matches any whitespace character `" \f\n\r\t\v"`
* `mpc_whitespaces` Matches zero or more whitespace characters
* `mpc_blank` Matches whitespaces and frees the result, returns `NULL`
* `mpc_newline` Matches `'\n'`
* `mpc_tab` Matches `'\t'`
* `mpc_escape` Matches a backslash followed by any character
* `mpc_digit` Matches any character in the range `'0'` - `'9'`
* `mpc_hexdigit` Matches any character in the range `'0'` - `'9'` as well as `'A'` - `'F'` and `'a'` - `'f'`
* `mpc_octdigit` Matches any character in the range `'0'` - `'7'`
* `mpc_digits` Matches one or more digit
* `mpc_hexdigits` Matches one or more hexdigit
* `mpc_octdigits` Matches one or more octdigit
* `mpc_lower` Matches and lower case character
* `mpc_upper` Matches any upper case character
* `mpc_alpha` Matches and alphabet character
* `mpc_underscore` Matches `'_'`
* `mpc_alphanum` Matches any alphabet character, underscore or digit
* `mpc_int` Matches digits and returns an `int*`
* `mpc_hex` Matches hexdigits and returns an `int*`
* `mpc_oct` Matches octdigits and returns an `int*`
* `mpc_number` Matches `mpc_int`, `mpc_hex` or `mpc_oct`
* `mpc_real` Matches some floating point number as a string
* `mpc_float` Matches some floating point number and returns a `float*`
* `mpc_char_lit` Matches some character literal surrounded by `'`
* `mpc_string_lit` Matches some string literal surrounded by `"`
* `mpc_regex_lit` Matches some regex literal surrounded by `/`
* `mpc_ident` Matches a C style identifier


Useful Parsers
--------------

* `mpc_startswith(mpc_parser_t *a);` Matches the start of input followed by `a`
* `mpc_endswith(mpc_parser_t *a, mpc_dtor_t da);` Matches `a` followed by the end of input
* `mpc_whole(mpc_parser_t *a, mpc_dtor_t da);` Matches the start of input, `a`, and the end of input  
* `mpc_stripl(mpc_parser_t *a);` Matches `a` striping any whitespace to the left
* `mpc_stripr(mpc_parser_t *a);` Matches `a` striping any whitespace to the right
* `mpc_strip(mpc_parser_t *a);` Matches `a` striping any surrounding whitespace
* `mpc_tok(mpc_parser_t *a);` Matches `a` and strips any trailing whitespace
* `mpc_sym(const char *s);` Matches string `s` and strips any trailing whitespace
* `mpc_total(mpc_parser_t *a, mpc_dtor_t da);` Matches the whitespace stripped `a`, enclosed in the start and end of input
* `mpc_between(mpc_parser_t *a, mpc_dtor_t ad, const char *o, const char *c);` Matches `a` between strings `o` and `c`
* `mpc_parens(mpc_parser_t *a, mpc_dtor_t ad);` Matches `a` between `"("` and `")"`
* `mpc_braces(mpc_parser_t *a, mpc_dtor_t ad);` Matches `a` between `"<"` and `">"`
* `mpc_brackets(mpc_parser_t *a, mpc_dtor_t ad);` Matches `a` between `"{"` and `"}"`
* `mpc_squares(mpc_parser_t *a, mpc_dtor_t ad);` Matches `a` between `"["` and `"]"`
* `mpc_tok_between(mpc_parser_t *a, mpc_dtor_t ad, const char *o, const char *c);` Matches `a` between `o` and `c`, where `o` and `c` have their trailing whitespace striped.
* `mpc_tok_parens(mpc_parser_t *a, mpc_dtor_t ad);` Matches `a` between trailing whitespace stripped `"("` and `")"`
* `mpc_tok_braces(mpc_parser_t *a, mpc_dtor_t ad);` Matches `a` between trailing whitespace stripped `"<"` and `">"`
* `mpc_tok_brackets(mpc_parser_t *a, mpc_dtor_t ad);` Matches `a` between trailing whitespace stripped `"{"` and `"}"`
* `mpc_tok_squares(mpc_parser_t *a, mpc_dtor_t ad);` Matches `a` between trailing whitespace stripped `"["` and `"]"`


Apply Functions
---------------

* `void mpcf_dtor_null(mpc_val_t *x);` Empty destructor. Does nothing
* `mpc_val_t *mpcf_ctor_null(void);` Returns `NULL`
* `mpc_val_t *mpcf_ctor_str(void);` Returns `""`
* `mpc_val_t *mpcf_free(mpc_val_t *x);` Frees `x` and returns `NULL`
* `mpc_val_t *mpcf_int(mpc_val_t *x);` Converts a decimal string `x` to an `int*`
* `mpc_val_t *mpcf_hex(mpc_val_t *x);` Converts a hex string `x` to an `int*`
* `mpc_val_t *mpcf_oct(mpc_val_t *x);` Converts a oct string `x` to an `int*`
* `mpc_val_t *mpcf_float(mpc_val_t *x);` Converts a string `x` to a `float*`
* `mpc_val_t *mpcf_escape(mpc_val_t *x);` Converts a string `x` to an escaped version
* `mpc_val_t *mpcf_escape_regex(mpc_val_t *x);` Converts a regex `x` to an escaped version
* `mpc_val_t *mpcf_escape_string_raw(mpc_val_t *x);` Converts a raw string `x` to an escaped version
* `mpc_val_t *mpcf_escape_char_raw(mpc_val_t *x);` Converts a raw character `x` to an escaped version
* `mpc_val_t *mpcf_unescape(mpc_val_t *x);` Converts a string `x` to an unescaped version
* `mpc_val_t *mpcf_unescape_regex(mpc_val_t *x);` Converts a regex `x` to an unescaped version
* `mpc_val_t *mpcf_unescape_string_raw(mpc_val_t *x);` Converts a raw string `x` to an unescaped version
* `mpc_val_t *mpcf_unescape_char_raw(mpc_val_t *x);` Converts a raw character `x` to an unescaped version

Fold Functions
--------------

* `mpc_val_t *mpcf_null(int n, mpc_val_t** xs);` Returns `NULL`
* `mpc_val_t *mpcf_fst(int n, mpc_val_t** xs);` Returns first element of `xs`
* `mpc_val_t *mpcf_snd(int n, mpc_val_t** xs);` Returns second element of `xs`
* `mpc_val_t *mpcf_trd(int n, mpc_val_t** xs);` Returns third element of `xs`
* `mpc_val_t *mpcf_fst_free(int n, mpc_val_t** xs);` Returns first element of `xs` and calls `free` on others
* `mpc_val_t *mpcf_snd_free(int n, mpc_val_t** xs);` Returns second element of `xs` and calls `free` on others
* `mpc_val_t *mpcf_trd_free(int n, mpc_val_t** xs);` Returns third element of `xs` and calls `free` on others
* `mpc_val_t *mpcf_strfold(int n, mpc_val_t** xs);` Concatenates all `xs` together as strings and returns result 
* `mpc_val_t *mpcf_maths(int n, mpc_val_t** xs);` Examines second argument as string to see which operator it is, then operators on first and third argument as if they are `int*`.


Case Study - Maths Language
===========================

Combinator Approach
-------------------

Passing around all these function pointers might seem clumsy, but having parsers be type-generic is important as it lets users define their own ouput types for parsers. For example we could design our own syntax tree type to use. We can also use this method to do some specific house-keeping or data processing in the parsing phase.

As an example of this power, we can specify a simple maths grammar, that ouputs `int *`, and computes the result of the expression as it goes along.

We start with a fold function that will fold two `int *` into a new `int *` based on some `char *` operator.

```c
mpc_val_t *fold_maths(int n, mpc_val_t **xs) {
  
  int **vs = (int**)xs;
    
  if (strcmp(xs[1], "*") == 0) { *vs[0] *= *vs[2]; }
  if (strcmp(xs[1], "/") == 0) { *vs[0] /= *vs[2]; }
  if (strcmp(xs[1], "%") == 0) { *vs[0] %= *vs[2]; }
  if (strcmp(xs[1], "+") == 0) { *vs[0] += *vs[2]; }
  if (strcmp(xs[1], "-") == 0) { *vs[0] -= *vs[2]; }
  
  free(xs[1]); free(xs[2]);
  
  return xs[0];
}
```

And then we use this to specify a basic grammar, which folds together any results.

```c
mpc_parser_t *Expr   = mpc_new("expr");
mpc_parser_t *Factor = mpc_new("factor");
mpc_parser_t *Term   = mpc_new("term");
mpc_parser_t *Maths  = mpc_new("maths");

mpc_define(Expr, mpc_or(2, 
  mpc_and(3, fold_maths,
    Factor, mpc_oneof("*/"), Factor,
    free, free),
  Factor
));

mpc_define(Factor, mpc_or(2, 
  mpc_and(3, fold_maths,
    Term, mpc_oneof("+-"), Term,
    free, free),
  Term
));

mpc_define(Term, mpc_or(2, mpc_int(), mpc_parens(Expr, free)));
mpc_define(Maths, mpc_whole(Expr, free));

/* Do Some Parsing... */

mpc_delete(Maths);
```

If we supply this function with something like `(4*2)+5`, we can expect it to output `13`.


Language Approach
-----------------

It is possible to avoid passing in and around all those function pointers, if you don't care what type is output by _mpc_. For this, a generic Abstract Syntax Tree type `mpc_ast_t` is included in _mpc_. The combinator functions which act on this don't need information on how to destruct or fold instances of the result as they know it will be a `mpc_ast_t`. So there are a number of combinator functions which work specifically (and only) on parsers that return this type. They reside under `mpca_*`.

Doing things via this method means that all the data processing must take place after the parsing. In many instances this is no problem or even preferable.

It also allows for one more trick. As all the fold and destructor functions are implicit, the user can simply specify the grammar of the language in some nice way and the system can try to build a parser for the AST type from this alone. For this there are a few functions supplied which take in a string, and output a parser. The format for these grammars is simple and familiar to those who have used parser generators before. It looks something like this.

```
number "number" : /[0-9]+/ ;
expression      : <product> (('+' | '-') <product>)* ;
product         : <value>   (('*' | '/')   <value>)* ;
value           : <number> | '(' <expression> ')' ;
maths           : /^/ <expression> /$/ ;
```

String literals are surrounded in double quotes `"`. Character literals in single quotes `'` and regex literals in slashes `/`. References to other parsers are surrounded in braces `<>` and referred to by name.

Parts specified one after another are parsed in order (like `mpc_and`), while parts separated by a pipe `|` are alternatives (like `mpc_or`). Parenthesis `()` are used to specify precedence. `*` can be used to mean zero or more of. `+` for one or more of. `?` for zero or one of. `!` for negation. And a number inside braces `{5}` to means so many counts of.

Rules are specified by rule name, optionally followed by an _expected_ string, followed by a colon `:`, followed by the definition, and ending in a semicolon `;`. The flags variable is a set of flags `MPCA_LANG_DEFAULT`, `MPCA_LANG_PREDICTIVE`, or `MPCA_LANG_WHITESPACE_SENSITIVE`. For specifying if the language is predictive or whitespace sensitive.

Like with the regular expressions, this user input is parsed by existing parts of the _mpc_ library. It provides one of the more powerful features of the library.

* * *

```c
mpc_parser_t *mpca_grammar(int flags, const char *grammar, ...);
```

This takes in some single right hand side of a rule, as well as a list of any of the parsers it refers to, and outputs a parser that does exactly what is specified by the rule.

* * *

```c
mpc_err_t* mpca_lang(int flags, const char *lang, ...);
```

This takes in a full language (zero or more rules) as well as any parsers referred to by either the right or left hand sides. Any parsers specified on the left hand side of any rule will be assigned a parser equivalent to what is specified on the right. On valid user input this returns `NULL`, while if there are any errors in the user input it will return an instance of `mpc_err_t` describing the issues.

* * *

```c
mpc_err_t* mpca_lang_file(int flags, FILE* f, ...);
```

This reads in the contents of file `f` and inputs it into `mpca_lang`.

* * *

```c
mpc_err_t* mpca_lang_contents(int flags, const char *filename, ...);
```

This opens and reads in the contents of the file given by `filename` and passes it to `mpca_lang`.


Error Reporting
===============

_mpc_ provides some automatic generation of error messages. These can be enhanced by the user, with use of `mpc_expect`, but many of the defaults should provide both useful and readable. An example of an error message might look something like this:

```
<test>:0:3: error: expected one or more of 'a' or 'd' at 'k'
```


Limitations & FAQ
=================

### Does _mpc_ support Unicode?

_mpc_ Only supports ASCII. Sorry! Writing a parser library that supports Unicode is pretty difficult. I welcome contributions!


### Is _mpc_ binary safe?

No. Sorry! Including NULL characters in a string or a file will probably break it. Just avoid it if possible.


### The Parser is going into an infinite loop!

While it is certainly possible there is an issue with _mpc_, it is probably the case that your grammar contains _left recursion_. This is something _mpc_ cannot deal with. _Left recursion_ is when a rule directly or indirectly references itself on the left hand side of a derivation. For example consider this left recursive grammar intended to parse an expression.

```
expr : <expr> '+' (<expr> | <int> | <string>);
```

When the rule `expr` is called, it looks the first rule on the left. This happens to be the rule `expr` again. So again it looks for the first rule on the left. Which is `expr` again. And so on. To avoid left recursion this can be rewritten as the following.

```
expr    : <int> <exprext> | <string> <exprext> ;
exprext : ('+' <expr>)? ;
```

Avoiding left recursion can be tricky, but is easy once you get a feel for it. For more information you can look on [wikipedia](http://en.wikipedia.org/wiki/Left_recursion) which covers some common techniques and more examples. Possibly in the future _mpc_ will support functionality to warn the user or re-write grammars which contain left recursion, but it wont for now.


### Backtracking isn't working!

_mpc_ supports backtracking, but will not completely backtrack up a parse tree if it encounters some success on the path it is going. To demonstrate this behaviour examine the following erroneous grammar, intended to parse either a C style identifier, or a C style function call.

```
factor : <ident>
       | <ident> '('  <expr>? (',' <expr>)* ')' ;
```

This grammar will never correctly parse a function call because it will always first succeed parsing the initial identifier. At this point it will encounter the parenthesis of the function call, give up, and throw an error. It will not backtrack far enough, to attempt the next potential option, which would have succeeded.

The solution to this is to always structure grammars with the most specific clause first, and more general clauses afterwards. This is the natural technique used for avoiding left-recursive grammars, so is a good habit to get into anyway.

```
factor : <ident> '('  <expr>? (',' <expr>)* ')'
       | <ident> ;
```

An alternative, and better option is to remove the ambiguity by factoring out the first identifier completely. This is better because it removes any need for backtracking at all! Now the grammar is predictive!

```
factor : <ident> ('('  <expr>? (',' <expr>)* ')')? ;
```


### How can I avoid the maximum string literal length?

Some compilers limit the maximum length of string literals. If you have a huge language string in the source file to be passed into `mpca_lang` you might encounter this. The ANSI standard says that 509 is the maximum length allowed for a string literal. Most compilers support greater than this. Visual Studio supports up to 2048 characters, while gcc allocates memory dynamically and so has no real limit.

There are a couple of ways to overcome this issue if it arises. You could instead use `mpca_lang_contents` and load the language from file or you could use a string literal for each line and let the preprocessor automatically concatenate them together, avoiding the limit. The final option is to upgrade your compiler. In C99 this limit has been increased to 4095.


### The automatic tags in the AST are annoying!

When parsing from a grammar, the abstract syntax tree is tagged with different tags for each primitive type it encounters. For example a regular expression will be automatically tagged as `regex`. Character literals as `char` and strings as `string`. This is to help people wondering exactly how they might need to convert the node contents.

If you have a rule in your grammar called `string`, `char` or `regex`, you may encounter some confusion. This is because nodes will be tagged with (for example) `string` _either_ if they are a string primitive, _or_ if they were parsed via your `string` rule. If you are detecting node type using something like `strstr`, in this situation it might break. One solution to this is to always check that `string` is the innermost tag to test for string primitives, or to rename your rule called `string` to something that doesn't conflict.

Yes it is annoying but its probably not going to change!


