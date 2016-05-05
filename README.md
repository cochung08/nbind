nbind
=====

Linux: [![build status](https://travis-ci.org/charto/nbind.svg?branch=master)](http://travis-ci.org/charto/nbind)
Windows: [![Build status](https://ci.appveyor.com/api/projects/status/xu5ooh1m3mircpde/branch/master?svg=true)](https://ci.appveyor.com/project/jjrv/nbind/branch/master)  
[![dependency status](https://david-dm.org/charto/nbind.svg)](https://david-dm.org/charto/nbind)
[![npm version](https://img.shields.io/npm/v/nbind.svg)](https://www.npmjs.com/package/nbind)

`nbind` is a bindings generator for Node.js plugins inspired by [embind](http://kripken.github.io/emscripten-site/docs/porting/connecting_cpp_and_javascript/embind.html) from [emscripten](http://emscripten.org). The bindings are built along with your C++ library without requiring any external generated code, using only C++11 templates and simple declarations.

To create bindings from a class X with a constructor taking 2 ints and a method Y with pretty much any kind of arguments and optional return value, you just need:

```
NBIND_CLASS(X) {
    construct<int, int>();

    method(Y);
}
```

Warning to git committers: rebase is used within feature branches (but not master).

Quick start
-----------

To compile and run some C++ code through JavaScript, just do:

```bash
npm install -g npm
git clone https://github.com/charto/nbind.git
cd nbind
npm install
npm test
```

You'll need to install Git, Node.js (0.10 or above) and a C++ compiler first. You can omit the first command if you have a recent npm (at least 2.7.0 should work, 2.14.0 is safer).

Working C++ compilers are at least:
- GCC 4.8 or above
- Clang 3.6 or above
- Visual Studio 2015 ([The Community version](https://www.visualstudio.com/en-us/products/visual-studio-community-vs.aspx) is fine)

Features
--------

`nbind` supports:
- Binding any number of C++ classes for use from Node.js, io.js or Electron.
- Exporting methods by just listing their names. Arguments and return types are autodetected.
- Types can be numbers, booleans, C strings or instances of other exported classes.
- A special functor `cbFunction` allows taking a JavaScript callback as a parameter and passing any number of supported types as arguments, casting the result to another supported type.
- Exported classes can be converted to "value types" by mapping them to a corresponding JavaScript constructor. The object is then automatically converted to C++ or JavaScript form as it's passed between the two languages.

Roadmap
-------

More is coming! Work is ongoing to:

- Compile the exact same C++ code and bindings with [Emscripten](http://emscripten.org/) to also run it in Chrome, Firefox or IE 10 and above.
- Automatically generate TypeScript `.d.ts` definition files from C++ code for IDE autocompletion and compile-time checks of JavaScript side code.

Example
-------

Let's look at a concrete example. Suppose we have a C++ class:

```C++
// test.cc part 1
// Get definition of nbind::cbFunction
#include "nbind/api.h"

class Test {

public:

    // Can be constructed with or without an initial state.
    Test(int newState = 42) : state(newState) {}

    // Getter and setter for private member.
    int getState() { return(state); }
    void setState(int newState) { state = newState; }

    // Call a callback function passing it the state variable and return the result.
    int callWithState(nbind::cbFunction &callback) {
        // The template argument defines what type to cast the return value to.
        return(callback.call<int>(state));
    }

    // Test returning a string from a static function.
    static const char *toString() { return("Test class"); }

private:

    int state;
};
```

Creating bindings for it is ridiculously easy, by adding the following in a source (not header) file:

```C++
// test.cc part 2
#include "nbind/BindingShort.h"

// Avoid errors if we're not compiling a Node.js module.
#ifdef NBIND_CLASS

NBIND_CLASS(Test) {
    // Both possible constructors, nbind will choose one based on how it's called.
    construct<>();
    construct<int>();

    // Getter and setter pair, nbind figures out it's a variable called "state".
    getset(getState, setState);

    // The methods, one taking a callback and the other static.
    method(callWithState);
    method(toString);
}

#endif
```

Then we can call it from JavaScript:

```javascript
// test.js
var nbind = require('nbind');
var pkg = nbind.module;

nbind.init(__dirname);

// Prints "Test class"
console.log('' + pkg.Test);

var obj = new pkg.Test();

// Prints 42
console.log(obj.state);

// Prints 43
console.log(++obj.state);

// Prints 86
console.log(obj.callWithState(function(state) {
	return(state * 2);
}));
```

Only a bit more configuration is needed to get everything installed, compiled and running.

`package.json` to list what needs to be installed and run:
```json
{
  "name": "nbind-demo",
  "scripts": {
    "install": "autogypi && node-gyp configure && node-gyp build",
    "test": "node test.js"
  },
  "dependencies": {
    "autogypi": "^0.1.0",
    "nbind": "^0.1.0"
  }
}
```

`autogypi.json` to list dependencies for autogypi and configure them automatically for node-gyp:
```json
{
	"dependencies": [
		"nbind"
	],
	"output": "auto.gypi"
}
```

`binding.gyp` to tell `node-gyp` what to do:
```json
{
	"targets": [
		{
			"includes": ["auto.gypi"],
			"sources": [
				"test.cc"
			]
		}
	]
}
```

That's it! Install, compile, test:
```bash
npm install
npm test
```

If it doesn't work, your `npm` might be too old. Try compiling again after running this command:

```bash
npm install -g npm
```

Usage and syntax
================

JavaScript side
---------------

`nbind` is intended to be used together with [autogypi](https://www.npmjs.com/package/autogypi) for easier dependency management. Declare it as a dependency in `autogypi.json`, run `autogypi` and include the resulting `auto.gypi` from your `binding.gyp` file. See the example section above for sample contents of these configuration files.

Your code might depend on other npm packages that also contain `nbind` exports. `autogypi` will make sure that when you run `node-gyp`, their sources will also get compiled and their includes will be in the include path when your code is compiled. When you run `node-gyp` it produces a module with all the exported classes from all packages you've included. Then you just do in the root directory of your package:

```javascript
var nbind = require('nbind');
nbind.init(__dirname);
```

This allows nbind to find the module you compiled. Afterwards, `nbind.module` will contain all the exported classes.

C++ nbind headers
-----------------

Use `#include "nbind/BindingShort.h"` at the end of your source file with only the bindings after it. The header defines macros with names like `construct` that are otherwise likely to break the code implementing your C++ class.

It's OK to include the file also when not targeting any JavaScript environment. `node-gyp` defines `BUILDING_NODE_EXTENSION` and Emscripten defines `EMSCRIPTEN` so when those are undefined, the include file does nothing.

Use `#include "nbind/api.h"` in your header files to use types in the `nbind` namespace if you need to report errors without throwing exceptions, or want to pass around callbacks or value objects.

You can use the `NBIND_ERR("message here");` macro to report an error before returning. It will be thrown as an error on the JavaScript side (C++ environments like Emscripten may not support throwing exceptions, but the JavaScript side code will).

Classes and constructors
------------------------

You can use an `#ifdef NBIND_CLASS` guard to skip your `nbind` export definitions when the includes weren't loaded.

The `NBIND_CLASS(className)` macro takes the name of your C++ class as an argument (without any quotation marks), and exports it to JavaScript using the same name. It's followed by a curly brace enclosed block of method exports, as if it was a function definition.

Constructors are exported with a macro call `construct<types...>();` where `types` is a comma-separated list of arguments to the constructor, such as `int, int`. Calling `construct` multiple times allows overloading it, but **each overload must have a different number of arguments**.

Methods and properties
----------------------

Methods are exported with a macro call `method(methodName);` which takes the name of the method as an argument (without any quotation marks). It gets exported to JavaScript with the same name. If the method is `static`, it becomes a property of the JavaScript constructor function and can be accessed like `className.methodName()`. Otherwise it becomes a property of the prototype and can be accessed like `obj = new className(); obj.methodName();`

Property getters are exported with a macro call `getter(getterName)` which takes the name of the getter method. `nbind` automatically strips a `get`/`Get`/`get_`/`Get_` prefix and converts the next letter to lowercase, so for example `getX` and `get_x` both would become getters of `x` to be accessed like `obj.x`

Property setters are exported together with getters using a macro call `getset(getterName, setterName)` which works much like `getter(getterName)` above. Both `getterName` and `setterName` are mangled individually so you can pair `getX` with `set_x` if you like. From JavaScript, `++obj.x` would then call both of them.

Callbacks and value objects
---------------------------

Callbacks can be passed to C++ methods by simply adding an argument of type `nbind::cbFunction &` to their declaration. They can be called with any number of any supported types without having to declare in any way what they accept. The JavaScript code will receive the parameters as JavaScript variables to do with them as it pleases. A callback argument `arg` can be called like `arg("foobar", 42);` in which case the return value is ignored. If the return value is needed, it must be called like `arg.call<type>("foobar", 42);` where `type` is the desired C++ type that the return value should be converted to.

Value objects are the most advanced concept supported so far. They're based on a `toJS` function on the C++ side and a `fromJS` function on the JavaScript side. Both receive a callback as an argument, and calling it with any parameters calls the constructor of the equivalent type in the other language (the bindings use [SFINAE](https://en.wikipedia.org/wiki/Substitution_failure_is_not_an_error) to detect the toJS function and turn the class into a value type). The callback on the C++ side is of type `nbind::cbOutput`. Value objects are passed **by value** on the C++ side to and from the exported function. `nbind` uses C++11 move semantics to avoid creating some additional copies on the way.

The equivalent JavaScript constructor must be registered on the JavaScript side by calling `nbind.bind('CppClassName', JSClassName);`, so `nbind` knows which types to translate between each other.

So if we have a file `test.cc` (can be tested by replacing the file with the same name in the example above):

```C++
#include "nbind/api.h"

int copies = 0;
int moves = 0;

// Coordinate pair, a value type we'll bind to an equivalent JavaScript type.

class Coord {

public:

    Coord(int x = 0, int y = 0) : x(x), y(y) {}

    // Let's count copy and move constructor usage.

    Coord(Coord &&other) : x(other.x), y(other.y) { ++copies; }

    Coord(const Coord &other) : x(other.x), y(other.y) { ++moves; }

    void addFrom(const Coord &other) {
        x += other.x;
        y += other.y;
    }

    // This is the magic function for nbind, which calls a constructor in JavaScript.

    void toJS(nbind::cbOutput output) {
        output(x, y);
    }

private:

    int x, y;
};

// This just sums coordinate pairs and returns them for testing.

class Accumulator {

public:

    Coord add(Coord other) {
        xy.addFrom(other);

        // a copy is made here.
        return(xy);
    }

    // This is a convenient place to read debug output, since we can't call methods of Coord
    // from JavaScript (it sees methods defined on the JavaScript object instead).

    int getCopies() { return(copies); }

    int getMoves() { return(moves); }

private:

    Coord xy;
};

#include "nbind/BindingShort.h"

#ifdef NBIND_CLASS

NBIND_CLASS(Coord) {
    construct<>();
    construct<int, int>();
}

NBIND_CLASS(Accumulator) {
    construct<>();

    method(add);
    getter(getCopies);
    getter(getMoves);
}

#endif
```

We can use it from a JavaScript file `test.js` like so:

```javascript
var nbind = require('nbind');

nbind.init(__dirname);

function Coord(x, y) {
    this.x = x;
    this.y = y;
}

Coord.prototype.fromJS = function(output) {
    output(this.x, this.y);
}

nbind.bind('Coord', Coord);

var accu = new nbind.module.Accumulator();

// Prints: { x: 2, y: 4 }
console.log(accu.add(new Coord(2, 4)));

// Prints: { x: 12, y: 34 }
console.log(accu.add(new Coord(10, 30)));

// Prints: 4 copies, 2 moves.
console.log(accu.copies + ' copies, ' + accu.moves + ' moves.');
```

License
=======

[The MIT License](https://raw.githubusercontent.com/charto/nbind/master/LICENSE)

Copyright (c) 2014-2016 BusFaster Ltd
