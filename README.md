*This repository is a mirror of the [component](http://component.io) module [fynyky/reactor.js](http://github.com/fynyky/reactor.js). It has been modified to work with NPM+Browserify. You can install it using the command `npm install npmcomponent/fynyky-reactor.js`. Please do not open issues or send pull requests against this repo. If you have issues with this repo, report it to [npmcomponent](https://github.com/airportyh/npmcomponent).*
Reactor.js
==========

Reactor is a lightweight library for [reactive programming](http://en.wikipedia.org/wiki/Reactive_programming). It provides reactive variables which automatically update themselves when the things they depend on change. This is similar to how a spreadsheet works, where cells can automatically change their own values in reaction to changes in other cells. 

Here's a quick example of what Reactor does:

```javascript
var foo = Signal(1);
var bar = Signal(function(){
  return foo() + 1;
});
var moo = Observer(function(){
  console.log(bar());
});

foo(); // 1
bar(); // 2

foo(6); // console logs 7

foo(); // 6
bar(); // 7
```

You declare how a variable should be calculated once, and it automatically recalculates itself when necessary. Observers get notified when the variables they use get updated. This makes it easy to keep a complex data model consistent, and a user interface up to date when a model is changed.

Reactor is designed to be unobtrusive and unopinionated. 

- There is no need to manually declare listeners or bindings. Reactor automatically keeps track of all that for you. 
- It imposes no particular structure on your code. Any variable can be easily replaced with a reactive one. 
- There is no need to learn special special syntax or use special classes, objects, or methods.

Overview
--------

Reactor has just 2 components: **signals** and **observers**

- **Signals** are values that are expected to change over time. Signals can depend on each other.
- **Observers** are functions which are triggered on signal changes.

A signal depends on another signal when it requires the other signal's value to determine its own. Similarly, an observer depends on a signal when it requires that signal's value to determine what to do.

Whenever a signal is updated, it automatically updates all its dependent signals & observers as well. Together, signals and observers form a graph representing the propagation of data throughout the application. Signals sit on the inside of the graph representing the internal data model, while observers are on the edges of the graph affecting external systems.

Here is an example of [simple note list application](http://jsfiddle.net/TjMgh/29/) using Reactor. A signal is used to represent user data while an observer is used to update the html.

```javascript
// The model is just an array of strings wrapped in a Signal
noteList = Signal(["sample note", "another sample note"]);

// The code to display the notes is wrapped in an Observer
// This code is automatically retriggered when the noteList is modified
Observer(function(){
  var noteListElement = document.getElementById("noteList");
  noteListElement.innerHTML = '';

  // "noteList().length" causes a read of noteList
  // This automatically sets noteList as a dependency of this Observer
  // When noteList is updated it automatically retriggers the whole code block
  for (var i = 0; i < noteList().length; i++) {
    var noteElement = document.createElement("div");
    noteElement.textContent = noteList()[i];
    noteListElement.appendChild(noteElement);
  }

});

// The input only needs to modify the Signal
// The UI update is automatically handled by the Observer
var noteInputElement = document.getElementById("noteInput");
noteInputElement.onkeydown = function(event){
  if (event.keyCode === 13) {
    noteList.push(noteInputElement.value);
    noteInputElement.value = '';
  }
};
```

Reactor adds very little overhead. You use `Signal` and `Observer` to wrap the appropriate variables and code blocks, swap reads and assignments for function calls, and you're good to go!

Comparison to other libraries
-----------------------------

Reactor is based on the same reactive principles as [Bacon.js](https://github.com/raimohanska/bacon.js) and [Knockout.js](http://knockoutjs.com/). The main difference is that Reactor is trying to be lightweight and keep the additional syntax and complexity to a minimum. Reactor sets dependencies for you automatically so there is no need to manually set subscriptions/listeners.

Compared to Knockout, Reactor does not provide semantic bindings directly to HTML. Instead, users set the appropriate HTML modifying functions as Observers.

Compared to Bacon, Reactor does not help to handle event streams.

Signals
-------

Signal are just values which other signals can depend on. 

Reactor provides a global function called `Signal`. It wraps a given value and returns it as a signal object.

```javascript
var foo = Signal(7);  
```

Signal objects are implemented as functions. To read the value of a signal, call it without any arguments.

```javascript
foo(); // returns 7
```

To change the value of a signal, pass it the new value as an argument.

```javascript
foo(9); // sets the signal's value to 9
```

Signals can take on any value: numbers, strings, booleans, and even arrays and objects.

```javascript
foo(2.39232);
foo("cheese cakes");
foo(true);
foo(["x", "y", "z"]);
foo({"moo": bar});
```

However, if a signal is given a function, it takes the return value of the function as its value instead of the function itself.

```javascript
var foo = Signal(function(){
  return 2 * 3;
});

foo(); // returns 6 instead of the function
```

Signals can have their value depend on other signals by using functions. If a different signal is read from within the given function, then that signal will automatically be set as a dependency. This means that when the dependency has been updated, the value of the dependent signal will be updated as well.

```javascript
var foo = Signal(7);
var bar = Signal(function(){ 
  return foo() * foo(); // since foo is read here, 
                        // is is registered as a dependency of bar
});

foo(); // returns 7 as expected
bar(); // returns 49 since it is defined as foo() * foo()

foo(10); // this updates the value of foo
         // and the value of bar as well

bar(); // returns 100 since it was automatically updated together with foo
```

Notice that there is no need to declare any listeners or bindings. Reactor automatically finds these dependencies in a signal's function definition.

This automatic updating allows signals to be linked together to form more complex dependency graphs. 

```javascript
var firstName = Signal("Gob");
var lastName = Signal("Bluth");

// fullName depends on both firstName and lastName
var fullName = Signal(function(){  
  return firstName() + " " + lastName();
});

// barbarianName depends only on firstName
var barbarianName = Signal(function(){
  return firstName() + " the Chicken"
});

// comicTitle depends on barbrianName and fullName and therefore
// indirectly depending on firstName and lastName
var comicTitle = Signal(function(){
  return "He who was once " + fullName() + " is now " + barbarianName();
});

firstName(); // "Gob"
lastName(); // "Bluth"
fullName(); // "Gob Bluth"
barbarianName(); // "Gob the Chicken"
comicTitle(); // "He who was once Gob Bluth is now Gob the Chicken"

firstName("Michael"); // updating firstname automatically updates 
                      // fullName, barbarianName, and comicTitle

firstName(); // "Michael"
lastName(); // "Bluth"
fullName(); // "Michael Bluth"
barbarianName(); // "Michael the Chicken"
comicTitle(); // "He who was once Michael Bluth is now Michael the Chicken"
```

As far as possible, signals should only read and avoid having any external effects. This refers to actions such as:

- Modifying the HTML
- Writing to disk
- Logging an interaction
- Triggering an alert

In a complex graph, a changed valued might cascade and cause some dependent signals' definitions to be invoked multiple times before propagation is complete.

In the example above, updating `firstName` first causes both `fullName` and `barbarianName` to update. This causes `comicTitle` to be updated twice. Once when `fullName` is updated, and again when `barbarianName` is updated. This is fine on its own since there are no external effects. 

However, if the definition of `comicTitle` included writing to disk then it would cause problems. Because `comicTitle` is updated twice, it would initiate 2 different writes. Furthermore, the first write would be incorrect as the change propagation would not have completed yet.

For external effects, it is recommended that observers are used instead.

Observers
---------

Observers are almost identical to signals except for 3 main differences:

- They are triggered only after all signals have been updated
- They are only triggered once per signal update
- Unlike signals, observers cannot be depended upon

Observers are used for external effects while signals are used for internal state. Signal functions might trigger multiple times before all signals have finished updating. If signals are used for external effects, they could triggered incorrectly and redundantly. Observers are triggered last and only once per update and therefore do not have this problem.

Observers are created in the same way as signals.

```javascript
var foo = Signal("random string");
var bar = Observer(function(){  // Triggers first on creation
  alert(foo());                 // Reads from foo and alerts "random string"
});                             // Is automatically registered as a dependent
                                // and triggers again whenever foo is changed
```

Just like signals, their dependencies are automatically calculated and triggered when the appropriate signal is updated.

```javascript
foo("a new random string"); // triggers bar which
                            // alerts "a new random string"
```

Just like signals, their functions can be updated.

```javascript
// change bar update the html instead of alerting
// triggers once immediately after updating
bar(function(){
  fooElement = document.getElementById("foo");
  fooElement.textContent = foo();
});

foo("this string will be logged now"); // triggers bar which now
                                       // logs the string instead
```

To disable an observer, pass in a null value.

```javascript
bar(null); // disables the observer 
```

Working with Arrays and Objects
-------------------------------

When updating Arrays and Objects, you should use Reactor's convenience methods instead of updating the objects directly. This means you should use:
- `foo.set(key, value)` instead of `foo()[key] = value`
- `foo.pop()` instead of `foo().pop()`
- `foo.push(value)` instead of `foo().push(value)`
- `foo.reverse()` instead of `foo().reverse()`
- `foo.shift()` instead of `foo().shift()`
- `foo.unshift(value)` instead of `foo().unshift(value)`
- `foo.sort(comparison)` instead of `foo().sort(comparison)`
- `foo.splice(start, length)` instead of `foo().splice(start, length)`

The reason for this is if a signal has an array as its value, directly updating the array will **not** update the signal's dependants. Because the signal object is still representing the same array, it does not detect the change. Instead, using the provided convenience methods does the same update but allows the change to be detected. This applies to objects as well.


```javascript
// foo initialized as a signal with an array as its value
var foo = Signal(["a", "b", "c"]); 

// bar initialized as a signal whos value depends on foo
var bar = Signal(function(){ 
  return foo().join("-");
});

foo(); // ["a","b","c"]
bar(); // "a-b-c"

// Updating foo's array directly does not trigger an update of bar
foo().push("d");
foo(); // ["a","b","c","d"]
bar(); // "a-b-c"

// Instead, updating using the convenience method does trigger the update of bar
foo.push("e");
foo(); // ["a","b","c","d","e"]
bar(); // "a-b-c-d-e"
```

Summary
-------

```javascript
var stringSignal = Signal("a string");    // Signals can be set to any value
var booleanSignal = Signal(true);
var numberSignal = Signal(1);

var dependentSignal = Signal(function(){  // If given a function, the signal's value 
                                          // is the return value of the function 
                                          // instead of the function itself
                                          
  return numberSignal() + 1;              // Reading from another signal automatically sets it
                                          // as a dependency
});                     

var stringSignal("a new string value");   // To update a signal just pass it a new value
                                          // this automatically updates all 
                                          // its dependents as well

var arraySignal = Signal([                // Signals can even be arrays or objects
  stringSignal,                           // which contain other signals
  booleanSignal,
  numberSignal
]);

var alertObserver = Observer(function(){  // Observers are just like signals except:
  alert(arraySignal().join(","));         // They are updated last
});                                       // They are only updated once per propagation
                                          // They cannot be depended on by signals

arraySignal.set(4, "a new value!")        // Convenience method for setting properties 
                                          // on an Array or Object Signal

arraySignal.push("foo");                  // Convenience methods for updating an array Signal
arraySignal.pop();
arraySignal.unshift("bar");
arraySignal.shift();
arraySignal.reverse();
arraySignal.sort();
arraySignal.splice(1, 2, "not a signal");
```
And if you like [Coffeescript](http://coffeescript.org/), Reactor gets even simpler!

```coffeescript

stringSignal = Signal "a string"                # Signals can be set to any value
booleanSignal = Signal true
numberSignal = Signal 1

dependentSignal = Signal -> numberSignal() + 1  # If given a function, the signal's value
                                                # is the return value of the function 
                                                # instead of the function itself
                                                # Reading from another signal automatically 
                                                # sets it as a dependency

stringSignal "a new string value"               # To update a signal just pass it a new value
                                                # this automatically updates all 
                                                # its dependents as well

arraySignal = Signal [                          # Signals can even be arrays or objects
  stringSignal                                  # which contain other signals
  booleanSignal
  numberSignal
]

alertObserver = Observer ->                     # Observers are just like signals except:
  alert arraySignal().join(",")                 # They are updated last
                                                # They are only updated once per propagation
                                                # They cannot be depended on by signals
                                                
arraySignal.set 4, "a new value!"               # Convenience method for setting properties 
                                                # on an object Signal

arraySignal.push "foo"                          # Convenience methods for updating an array Signal      
arraySignal.pop()
arraySignal.unshift("bar")
arraySignal.shift()
arraySignal.reverse()
arraySignal.sort()
arraySignal.splice 1, 2, "not a signal"
```
Installation and use
--------------------

Download [reactor.js](https://github.com/fynyky/reactor.js/raw/master/reactor.js) and include it in your application. 

Reactor has just 2 components: `Signal` and `Observer`
- For browsers, they will be bound to window as global objects for use anywhere.
- For node.js, they will be bound as properties of the exports object to be imported as modules

For [Coffeescript](http://coffeescript.org/) users, you can instead use [reactor.coffee](https://github.com/fynyky/reactor.js/raw/master/reactor.coffee) for your coffeescript workflow.

For [node.js](http://nodejs.org/) users, Reactor.js is also [available on npm](https://npmjs.org/package/reactorjs) by running
```
$ npm install reactorjs
```
And importing it into your application by adding
```javascript
Reactor = require("reactorjs");
Signal = Reactor.Signal;
Observer = Reactor.Observer;
```
For the lucky people using both coffeescript and node.js: you can just use
```coffeescript
{Signal, Observer} = require "reactorjs"
```
