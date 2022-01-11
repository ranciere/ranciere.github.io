---
layout: post
title:  "zoltan: a minimalist Lua binding"
image:
  path: /docs/assets/images/zoltan.png
  thumbnail: /docs/assets/images/zoltan.png
---

[`zoltan`](https://github.com/ranciere/zoltan) is an open source, Sol-inspired minimalist Lua binding library for Zig.

### Why?
I've always been curious about programming languages. My first favorite was C++ template meta-programming, more than 10 years ago. Then I discovered Lua, which impressed me with its simplicity and a brilliant trick: there is only one composed type that exists in Lua, the `table`. You can use it as an array, map, object, for everything!

From the [Lua site](https://www.lua.org/about.html): "Lua is a powerful, efficient, lightweight, embeddable scripting language. It supports procedural programming, object-oriented programming, functional programming, data-driven programming, and data description."

Some use-cases:
- Dynamic configuration of the application (you can handle complex configuration cases which involve logic)
- To support user-defined extensions
- To automate repetitive user tasks (Neovim)
- To develop UI business logic (a lot of games do this)

Last year during Hacker News surfing I discovered Zig, and it had immediately aroused my interest: a simple, but powerful language; and it has also a brilliant trick, the [`comptime`](https://kristoff.it/blog/what-is-zig-comptime/). 

So I decided I would develop a Lua binding library in Zig during my sabbatical. It would be a gentle & productive introduction to Zig, furthermore involve meta-programming and Lua. The best combo! :)

### Usage
[`zoltan`](https://github.com/ranciere/zoltan) supports the most important use-cases:

- Creating Lua engine
- Running Lua code
- Handling Lua globals
- Handling Lua tables
- Calling functions from both side
- Defining user types

First, we create a Lua engine:
```zig
const Lua = @import("lua").Lua;
...
pub fn main() anyerror!void {
  ...
  var lua = try Lua.init(std.testing.allocator);
  defer lua.destroy(); // it will call lua_close() at the end
```
Open standard Lua libs to support `print`:
```zig
lua.openLibs();
```
Set global variable:
```zig
lua.set("meaning_of_life", 42);
```
Run Lua code:
```zig
lua.run("print('meaning_of_life')");  // Prints '42'
```
Get global variable:
```zig
var meaning = try lua.get(i32, "meaning_of_life");
```
Create table:
```zig
var tbl = try lua.createTable();
defer lua.release(tbl);                   // It has to be released
```
Store table as global:
```zig
lua.set("tbl", tbl);
```
Get/set table member:
```zig
tbl.set(42, "meaning of life");            // Set by numeric key
tbl.set("meaning_of_life", 42);           // Set by string key
_ = try tbl.get([] const u8, 42);          // Get by numeric key
_ = try tbl.get(i32, "meaning_of_life");  // Get by numeric key
```
Call Zig function from Lua:
```zig
fn sum(a: i32, b: i32) i32 {
  return a+b;
}
...
// Set as global
lua.set("sum", sum);
lua.run("print(sum(1,1))");               // Prints '2'
// Set as table member
tbl.set("sum", sum);
lua.run("print(tbl.sum(tbl.meaning_of_life,0))");  // Prints '42'
```
Call Lua function from Zig:
```zig
lua.run("function lua_sum(a,b) return a+b; end"); 
var luaSum = lua.getResource(Lua.Function(fn(a: i32, b: i32) i32), "lua_sum");
defer lua.release(luaSum);                // Release the reference later
var res = luaSum.call(.{3,3});            // res == 6
```
Define and use user types:
```zig
const CheatingCalculator = struct {
  offset: i32 = undefined,

  pub fn init(_offset: i32) CheatingCalculator {
    return CheatingCalculator{ .offset = _offset, };
  }

  pub fn destroy(_: *CheatingCalculator) void { }

  pub fn add(self: *CheatingCalculator, a: i32, b: i32) i32 {
    return self.offset + a + b;
  }

  pub fn sub(self: *CheatingCalculator, a: i32, b: i32) i32 {
    return self.offset + a - b;
  }
};
...
try lua.newUserType(CheatingCalculator);

const cmd = 
  \\calc = CheatingCalculator.new(42)
  \\print(calc, getmetatable(calc))  -- prints 'CheatingCalculator: 0x7f59fdd79a78	table: 0x7f59fdd796c0'
  \\print(calc:add(1,1), calc:sub(10, 1)) -- prints: '44	51'
;
lua.run(cmd);

//// OR from Zig
var calc = try lua.createUserType(CheatingCalculator, .{42});
defer lua.release(calc);

var res0 = calc.ptr.add(1, 1);  // == 44
var res1 = calc.ptr.sub(10, 1); // == 51
```

### Internals
Lua - as it is embeddable by design - has a well-documented, orthogonal, stack-based C API. `zoltan` transform this plain C API to Zig flavor.

#### Lua instance
A `lua_State` represents a Lua engine. In `zoltan`, an instance of the `Lua` struct holds the reference of the corresponding `lua_State`, and it's user-data. This user-data contains the provided allocator and the registered type map.

#### Type matching
The type set of Lua is kept to a minimum: bool, integer, number, string, table, function, userdata. The following table contains the matching between Lua and Zig:

| **Lua**    | **Zig**                 |
|------------|-------------------------|
| `string`   | `[] const u8`           |
| `integer`  | `i16`, `i32`, `i64`     |
| `number`   | `f32`, `f64`            |
| `boolean`  | `bool`                  |
| `table`    | `Lua.Table`             |
| `function` | `Lua.Function`          |
| `userdata` | `Lua.Ref`               |

The instances of registered user types become Lua userdata with appropriate [metatable](https://www.lua.org/pil/13.html).

#### Lua stack

The Lua API is stack-based, every operation must be performed via the stack. For example, to call a Lua function you have to push all of the arguments on the stack, and after the function returns the result should be popped. Similarly, setting a table's key to a value requires three elements on the stack:  the (1) table, the (2) key, and the (3) value.

Besides, Lua's inner variant representation is hidden therefore - unlike strings and scalars - getting a pointer to a function or a table is impossible. But somehow we must handle these types from the C side, what can we do? Lua's answer is its _Registry_ and _Reference system_ (which is a special table). You can register Lua objects and refer them later via stack operations. 

It is already clear from these simple examples, that the most important features of a Lua binding library are the generic `push` and `pop` operations. And this is the point, where the meta-programming comes into play :)

#### Push
The first few lines of `push`:
```zig
fn push(L: *lualib.lua_State, value: anytype) void {
  const T = @TypeOf(value);
  switch (@typeInfo(T)) {
    .Bool => lualib.lua_pushboolean(L, @boolToInt(value)),
```
The function takes the Lua engine and a value. The principle of operation is very simple: it switches by the type of the value and then executes the type matching strategy: 

- the scalars and string cases are straightforward (eg. `lua_pushinteger`, `lua_pushboolean` etc.)
- in case of Lua objects, it will push the reference number (see _pop_)

The interesting thing comes when we push a Zig function on the stack; for this, we use the [C Closure](https://www.lua.org/manual/5.3/manual.html#4.4) functionality of Lua. 
First, we create a type with the [`ZigCallHelper`](https://github.com/ranciere/zoltan/blob/66297372eb75e35fa9f97e9ce4b962386cc93d71/src/lua.zig#L601-L675) generic method, based on the footprint of the function:
```zig
const Helper = ZigCallHelper(@TypeOf(value));
Helper.pushFunctor(L, value) catch unreachable;
```
The `Helper` implements the tasks of the function call:

- preparing arguments (popping from the stack based on the types of the input arguments)
- calling the method
- pushing the result
- destroying the allocated arguments during the preparing phase

After creating `Helper`, we execute its `pushFunctor`, which performs the following:
- pushes the address of the function
- pushes the address of a C ABI compatible [_Zig closure_](https://github.com/ranciere/zoltan/blob/66297372eb75e35fa9f97e9ce4b962386cc93d71/src/lua.zig#L657-L671), which will execute the call using the helpers mentioned above.

#### Pop
In Zig every operation is explicit, there are no hidden control flow or memory allocations. In practice, this means that the functions which acquire resources must be distinguished in some way (eg. by naming currently).

Because in some cases acquiring resource is required during the _pop_ operation, there are types of _pop_: 
- plain `pop` which is basically can be used in the case of scalars and strings
- `popResource` can be used in case of dynamic arrays and Lua objects (Table, Function, user types). In the latter case, the objects are [registered](https://www.lua.org/manual/5.3/manual.html#4.5) and later referenced by the resulting ID.

Similar to the _push_, most of the implementation of pop functions is quite straightforward, except the `LuaFunction`. 
It refers the corresponding Lua object (the wrapped function) and provides a `call` method. `call` get the object via the reference id, then pushes all of the input arguments, calls the function and pop the result.

### My impressions of Zig

Starting from scratch, without any prior knowledge it took about three weeks of active work to develop and test `zoltan`. Of course, I've serious experience with various programming languages, but it is still very impressive. Zig fulfilled its promise: it is absolutely easy to learn; (almost) everything is clear. I've only experienced two oddities (which I can't judge yet if these are good or bad): the lack of RAII (destructors) and the mode of [manipulating types](https://ikrima.dev/dev-notes/zig/zig-metaprogramming/).

#### Lack of RAII (and destructors)

In C++ the main strategy to prevent resource leak is using _Resource acquisition is initialization_ (RAII) idiom. The main drawbacks of this approach are the hidden control flow and forcing everything to become a class. Zig's approach is completely different: it uses the `defer` keyword to postpone the clean-up to the end of the scope. In many cases during the development, I reflexively wanted to use some kind of RAII (for example `std::unique_ptr`) when I realized that this was not possible here. Although it required a different way of thinking, I was eventually able to solve everything what I wanted. 
However, I feel that managing **shared** memory/resources (`std::shared_ptr`) could be problematic; and hard-to-maintain code may emerge in the future.

#### Manipulating types in compile time

After many years of active template meta-programming in C++ (and with some functional programming knowledge), Zig's approach of type manipulation was a busting, ambiguous experience. 
At first, I felt like a butcher. Butcher of types. I could handle everything easily, all C++ template tricks ([SFINAE](https://en.wikipedia.org/wiki/Substitution_failure_is_not_an_error), [concepts](https://en.wikipedia.org/wiki/Concepts_(C%2B%2B))), are perfectly simplified. On the other hand, I felt a little insecure: I missed the forced, mathematically proven correctness. 

But after a while, all my doubts were gone: using Zig for meta-programming is a pleasant experience; it's like _I can script the C++ compiler_.
You don't have to debug exotic compiler bugs, you can work very efficiently and focus on your real job.

