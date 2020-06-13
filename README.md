# Quoted Default Arguments

Goal: Facilitate implicit forwarding of function parameters by executing quoted expressions or types in function or template parameter default arguments. Extend the function declaration and function call syntax with a semicolon argument delimiter to allow defaulted parameters to trail function and template parameter packs.

## Quotations

```cpp
void func(int x, int y, logger_t& logger = `logger`, allocator& alloc = `alloc`);

void foo(logger_t& logger, allocator& alloc) {
  // Arguments for logger and alloc are implicitly bound to the parameters of
  // foo with those names.
  func(10, 20);
}
```

A quoted default argument (QDA) is the default argument of a function or template parameter that is evaluated at the point of the function call, with full access to the symbols defined there. A function participating in QDA establishes a convention on what names to expect, and the caller must provide symbols with those names, or provide an explicit argument. 

```cpp
void func(int x, int y, logger_t& logger = `logger`, allocator& alloc = `alloc`);

void foo(logger_t& logger, allocator& alloc) {
  // logger = logger.
  func(10, 20);

  // Provide arguments to override defaults.
  logger_t logger2;
  func(30, 40, logger2);
}
```

Objections on the grounds of code hygiene are anticipated. It is possible to restrict the QDA to only binding to function parameters at the point of call, making this purely an argument-forwarding mechanism. 

However it doesn't take much imagination to come up with QDAs that use simple expressions:

```cpp
void write_log(const char* msg, const char* f = `__builtin_FILE()`, 
  unsigned ln = `__builtin_LINE()`);

// f and ln are set to the source locations at the point of the call, not at
// the point of the default argument declarations.
write_log("Defcon 4");
```

We're designing programming languages, so I favor making new features programmable.

## Parameter packs

These more potent default arguments should be made with compatible parameter packs. The easiest way to do this is to overhaul the way functions are declared and called. Consider supporting a semicolon delimiter anywhere in a function's parameter list. Passing arguments to the function requires matching semicolons before proceeding to the next function parameter.

```cpp
template<typename... types_t, typename logger_t>
void write_log(const types_t&... args; logger_t& logger = `logger`);

write_log(1, 2, 3); # 1
write_log(1, 2, 3; tiktok_logger); # 2
```

Without the semicolon in #2, `tiktok_logger` would be assigned to the fourth element of the parameter pack `args`, and `logger` would be bound to its QDA argument.

Use of the semicolon signals the caller's intent to terminate the parameter pack. The next argument binds to the next parameter. 

```cpp
template<typename... pack1_t, typename... pack2_t, typename... pack3_t>
void get_3_pack(pack1_t... pack1, pack2_t... pack2, pack3_t... pack3);

get_3_pack(1, 2, 3; ; 4, 5); // pack1 = {1, 2, 3}. pack2 = { }. pack3 = {4, 5}.
```

This fixes a long-standing issue with C++, which is deduction of multiple parameter packs from function arguments: you cannot do it. You can deduce multiple parameter packs given multiple template specializations in the declaration, but that's an advanced hack requiring tuple intermediates and hard-to-reason workarounds.

This also allows default arguments in the middle of a function, allowing more thematic grouping of parameters.

This looks like a really fundamental language change, but it would require an almost superficial amendment to the specification. Argument delimiters are part of the function declaration but are not part of its type. Forming a function pointer/reference doesn't require these delimiters, because it can't do anything with them: the number and types of parameters have already been decided when forming the pointer, so the caller is obligated to provide all parameters explicitly. Access to default arguments and argument delimiters was lost when the name of the function was replaced by a pointer to the function.

You cannot overload functions on argument delimiters, and all redeclarations must have the same delimiter structure or the program is ill-formed.

When performing overload resolution, argument delimiters are evaluated early: Overloads with an delimiter structure that's incompatible with the delimiters specified by the caller are removed the candidate set. Similar logic is currently performed when checking if the argument count is compatible with the parameter count (taking into account parameter packs, default arguments and C-style ellipses). The extension to argument delimiters simply means doing this existing check for each delimited group of arguments and parameters.

There's a similar concern when dealing with [[temp.deduct.funcaddr]](http://eel.is/c++draft/temp.deduct.funcaddr). This operation attempts to choose the best viable candidate in an overload set given a function or function pointer type. The function type we're converting to has no delimiter information. The candidates we're converting from have parameter packs and default arguments. We must use the argument delimiters in those candidates to coordinate parameter types from the specified function type with pack elements in the function itself.

## Argument deduction with template args.

```cpp
template<typename type_t>
void func(type_t x = 5);

int main() {
  func(); // Ill-formed! Cannot deduce type_t.
}
```

C++ doesn't allow argument deduction from function parameters' default arguments. That should be fixed to make QDA more useful. Both QDA and ordinary default arguments should be evaluated during argument deduction if the template parameter cannot be otherwise deducedx. If default argument substitution fails, kick the overload out of the candidate set (it would have been booted anyways).

## Optional parameters.

A more radical feature is to allow optional QDAs. Consider a library function that supports implicit forwarding of loggers and allocators. Ideally this code could be used when one or neither of these objects are available to the caller.

```cpp
void func(int x, int y, auto& logger ? `logger`, auto& alloc ? `alloc`);

allocator_t alloc;
func(5, 6);   // An alloc is available, but name lookup for logger fails.
```

This is a function template with placeholder parameter types. We can optionally put concepts on for constrained placeholder parameter types. But they still must be deduced when binding to default arguments. What if substitution of the default argument fails?

If the parameter is an _optional parameter_, meaning it's trailed by the `?` token, default argument substitution failure _is a successful_ deduction. The corresponding parameter is deduced as _undefined_. Undefined parameters have the lowest rank for the purpose of overload resolution, so a candidate with a default argument that successfully substitutes would always be chosen over a candidate with a substitution failure.

Undefined parameters do not contribute to the function's type. The function type of the instantiated `func` above is `void(int, int, allocator_t&)`. Undefined parameters are, however, part of the function's definition and their access must be guarded against by _if-defined_ constructs.

```cpp
void func(int x, int y, auto& logger ? `logger`, auto& alloc ? `alloc`) {
  if defined(logger) {
    logger<< "Got parameters "<< x<< " and "<< y;
  }

  int* storage = nullptr;
  if defined(alloc) {
    if defined(logger) {
      logger<< "Got allocator "<< alloc.name();
    }

    storage = alloc.allocate(sizeof(double));

  } else 
    storage = operator new(sizeof(double));
}
```

`defined` is a contextual keyword. It could also be used on the type of the parameter, when that parameter has a name. It could also be replaced by an `if constexpr(requires { logger; })`, but that usage is less suggestive of intent.

## To backtick or not to backtick?

I immediately reached for backticks to mark QDAs. Quote/quasiquote capability exists in languages like Racket and Terra. I don't actually know what the feature does in those languages. However it involves code inside backticks that gets expanded somewhere else. It's unhygienic by C++ standards, but it's not C++ so it's okay.

Andrei suggested using the default keyword and dropping the backticks. I like deploying that keyword for this context. I'm not wedded to backticks either. But we do require some kind of parenthesis/brace/bracket/tick to group these QDA tokens together.

C++ is not a context-free grammar. Consider trying to parse this QDA:

```cpp
void hell_function(auto x default a < b , c > :: y);
```

As a human you see this as qualified name lookup into a template specialization `a<b, c>`. But the compiler can't actually parse this until `hell_function` is invoked. When encountering the comma, the compiler doesn't know whether to start a new parameter declaration or not. It requires name lookup of the symbol `a` to determine if the following `<` is the start of a _template-argument-list_ or just an arithmetic less-than. A traditional default argument would be permitted name lookup at definition* (the real story is more complicated and fills me with rage) so the compiler knows how to interpret that `<`, and by extension the comma.

It may be cool to put dump those tokens into brackets, which don't add visual noise like an additional set of parentheses would:

```cpp
void heaven_function(auto x default [a<b, c>::y]);
```

Now parsing the function definition only requires marking the range of tokens inside the QDA's brackets.

