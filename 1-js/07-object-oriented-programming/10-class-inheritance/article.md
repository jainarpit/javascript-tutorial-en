
# Class inheritance, super

Classes can extend one another. There's a nice syntax, technically based on the prototypal inheritance.

To inherit from another class, we should specify `"extends"` and the parent class before the brackets `{..}`.

[cut]

Here `Rabbit` inherits from `Animal`:

```js run
class Animal {

  constructor(name) {
    this.speed = 0;
    this.name = name;
  }

  run(speed) {
    this.speed += speed;
    alert(`${this.name} runs with speed ${this.speed}.`);
  }

  stop() {
    this.speed = 0;
    alert(`${this.name} stopped.`);
  }

}

*!*
// Inherit from Animal
class Rabbit extends Animal {
  hide() {
    alert(`${this.name} hides!`);
  }
}
*/!*

let rabbit = new Rabbit("White Rabbit");

rabbit.run(5); // White Rabbit runs with speed 5.
rabbit.hide(); // White Rabbit hides!
```

The `extends` keyword actually adds a `[[Prototype]]` reference from `Rabbit.prototype` to `Animal.prototype`, just as you expect it to be, and as we've seen before.

![](animal-rabbit-extends.png)

So now `rabbit` has access both to its own methods and to methods of `Animal`.

````smart header="Any expression is allowed after `extends`"
Class syntax allows to specify not just a class, but any expression after `extends`.

For instance, a function call that generates the parent class:

```js run
function f(phrase) {
  return class {
    sayHi() { alert(phrase) }
  }
}

*!*
class User extends f("Hello") {}
*/!*

new User().sayHi(); // Hello
```
Here `class User` inherits from the result of `f("Hello")`.

That may be useful for advanced programming patterns when we use functions to generate classes depending on many conditions and can inherit from them.
````

## Overriding a method

Now let's move forward and override a method. As of now, `Rabbit` inherits the `stop` method that sets `this.speed = 0` from `Animal`.

If we specify our own `stop` in `Rabbit`, then it will be used instead:

```js
class Rabbit extends Animal {
  stop() {
    // ...this will be used for rabbit.stop()
  }
}
```


...But usually we don't want to totally replace a parent method, but rather to build on top of it, tweak or extend its functionality. We do something in our method, but call the parent method before/after it or in the process.

Classes provide `"super"` keyword for that.

- `super.method(...)` to call a parent method.
- `super(...)` to call a parent constructor (inside our constructor only).

For instance, let our rabbit autohide when stopped:

```js run
class Animal {

  constructor(name) {
    this.speed = 0;
    this.name = name;
  }

  run(speed) {
    this.speed += speed;
    alert(`${this.name} runs with speed ${this.speed}.`);
  }

  stop() {
    this.speed = 0;
    alert(`${this.name} stopped.`);
  }

}

class Rabbit extends Animal {
  hide() {
    alert(`${this.name} hides!`);
  }

*!*
  stop() {
    super.stop(); // call parent stop
    this.hide(); // and then hide
  }
*/!*
}

let rabbit = new Rabbit("White Rabbit");

rabbit.run(5); // White Rabbit runs with speed 5.
rabbit.stop(); // White Rabbit stopped. White rabbit hides!
```

Now `Rabbit` has the `stop` method that calls the parent `super.stop()` in the process.

````smart header="Arrow functions have no `super`"
As was mentioned in the chapter <info:arrow-functions>, arrow functions do not have `super`.

If accessed, it's taken from the outer function. For instance:
```js
class Rabbit extends Animal {
  stop() {
    setTimeout(() => super.stop(), 1000); // call parent stop after 1sec
  }
}
```

The `super` in the arrow function is the same as in `stop()`, so it works as intended. If we specified a "regular" function here, there would be an error:

```js
// Unexpected super
setTimeout(function() { super.stop() }, 1000);
```
````


## Overriding constructor

With constructors, things are is a little bit tricky.

Till now, `Rabbit` did not have its own `constructor`.

According to the [specification](https://tc39.github.io/ecma262/#sec-runtime-semantics-classdefinitionevaluation), if a class extends another class and has no `constructor`, then the following `constructor` is generated:

```js
class Rabbit extends Animal {
  // generated for extending classes without own constructors
*!*
  constructor(...args) {
    super(...args);
  }
*/!*
}
```

As we can see, it basically calls the parent `constructor` passing it all the arguments. That happens if we don't write a constructor of our own.

Now let's add a custom constructor to `Rabbit`. It will specify the `earLength` in addition to `name`:

```js run
class Animal {
  constructor(name) {
    this.speed = 0;
    this.name = name;
  }
  // ...
}

class Rabbit extends Animal {

*!*
  constructor(name, earLength) {
    this.speed = 0;
    this.name = name;
    this.earLength = earLength;
  }
*/!*

  // ...
}

*!*
// Doesn't work!
let rabbit = new Rabbit("White Rabbit", 10); // Error: this is not defined.
*/!*
```

Whoops! We've got an error. Now we can't create rabbits. What went wrong?

The short answer is: constructors in inheriting classes must call `super(...)`, and (!) do it before using `this`.

...But why? What's going on here? Indeed, the requirement seems strange.

Of course, there's an explanation. Let's get into details, so you'd really understand what's going on.

In JavaScript, there's a distinction between a "constructor function of an inheriting class" and all others. In an inheriting class, the corresponding constructor function is labelled with a special internal property `[[ConstructorKind]]:"derived"`.

The difference is:

- When a normal constructor runs, it creates an empty object as `this` and continues with it.
- But when a derived constructor runs, it doesn't do it. It expects the parent constructor to do this job.

So if we're making a constructor of our own, then we must call `super`, because otherwise the object with `this` reference to it won't be created. And we'll get an error.

For `Rabbit` to work, we need to call `super()` before using `this`, like here:

```js run
class Animal {

  constructor(name) {
    this.speed = 0;
    this.name = name;
  }

  // ...
}

class Rabbit extends Animal {

  constructor(name, earLength) {
*!*
    super(name);
*/!*
    this.earLength = earLength;
  }

  // ...
}

*!*
// now fine
let rabbit = new Rabbit("White Rabbit", 10);
alert(rabbit.name); // White Rabbit
alert(rabbit.earLength); // 10
*/!*
```


## Super: internals, [[HomeObject]]

Let's get a little deeper under the hood of `super`. We'll see some interesting things by the way.

First to say, from all that we've learned till now, it's impossible for `super` to work.

Yeah, indeed, let's ask ourselves, how it could technically work? When an object method runs, it gets the current object as `this`. If we call `super.method()` then, how to retrieve that  method? In other words, we need to take the `method` from the parent prototype of the current object. How, technically, we (or a JavaScript engine) can do it?

Maybe we can get it `[[Prototype]]` of `this`, as `this.__proto__.method`? Unfortunately, that won't work.

Let's try to do it. Without classes, using plain objects for the sake of simplicity.

Here, `rabbit.eat()` should call `animal.eat()` method of the parent object:

```js run
let animal = {
  name: "Animal",
  eat() {
    alert(this.name + " eats.");
  }
};

let rabbit = {
  __proto__: animal,
  name: "Rabbit",
  eat() {
*!*
    this.__proto__.eat.call(this); // (*)
*/!*
  }
};

rabbit.eat(); // Rabbit eats.
```

At the line `(*)` we take `eat` from the prototype (`animal`) and call it in the context of the current object. Please note that `.call(this)` is important here, because a simple `this.__proto__.eat()` would execute parent `eat` in the context of the prototype, not the current object.

And here it works.

Now let's add one more object to the chain. We'll see how things break:

```js run
let animal = {
  name: "Animal",
  eat() {
    alert(this.name + " eats.");
  }
};

let rabbit = {
  __proto__: animal,
  eat() {
    // ...bounce around rabbit-style and call parent (animal) method
    this.__proto__.eat.call(this); // (*)
  }
};

let longEar = {
  __proto__: rabbit,
  eat() {
    // ...do something with long ears and call parent (rabbit) method
    this.__proto__.eat.call(this); // (**)
  }
};

*!*
longEar.eat(); // Error: Maximum call stack size exceeded
*/!*
```

The code doesn't work any more! We can see the error trying to call `longEar.eat()`.

It may be not that obvious, but if we trace `longEar.eat()` call, then we can see why. In both lines `(*)` and `(**)` the value of `this` is the current object (`longEar`). That's essential: all object methods get the current object as `this`, not a prototype or something.

So, in both lines `(*)` and `(**)` the value of `this.__proto__` is exactly the same: `rabbit`. They both call `rabbit.eat` without going up the chain.

In other words:

1. Inside `longEar.eat()`, we pass the call up to `rabbit.eat` giving it the same `this=longEar`.
    ```js
    // inside longEar.eat() we have this = longEar
    this.__proto__.eat.call(this) // (**)
    // becomes
    longEar.__proto__.eat.call(this)
    // or
    rabbit.eat.call(this);
    ```
2. Inside `rabbit.eat`, we want to pass the call even higher in the chain, but `this=longEar`, so `this.__proto__.eat` is `rabbit.eat`!

    ```js
    // inside rabbit.eat() we also have this = longEar
    this.__proto__.eat.call(this) // (*)
    // becomes
    longEar.__proto__.eat.call(this)
    // or (again)
    rabbit.eat.call(this);
    ```

3. ...So `rabbit.eat` calls itself in the endless loop, because it can't ascend any further.

![](this-super-loop.png)

There problem is unsolvable, because `this` must always be the calling object itself, no matter which parent method is called. So its prototype will always be the immediate parent of the object. We can't go up the chain.

### `[[HomeObject]]`

To provide the solution, JavaScript adds one more special internal property for functions: `[[HomeObject]]`.

**When a function is specified as a class or object method, its `[[HomeObject]]` property becomes that object.**

This actually violates the idea of "unbound" functions, because methods remember their objects. And `[[HomeObject]]` can't be changed, so this bound is forever. So that's a very important change in the language.

But this change is safe. `[[HomeObject]]` is used only for calling parent methods in `super`, to resolve the prototype. So it doesn't break compatibility.

Let's see how it works for `super` -- again, using plain objects:

```js run
let animal = {
  name: "Animal",
  eat() {         // [[HomeObject]] == animal
    alert(this.name + " eats.");
  }
};

let rabbit = {
  __proto__: animal,
  name: "Rabbit",
  eat() {         // [[HomeObject]] == rabbit
    super.eat();
  }
};

let longEar = {
  __proto__: rabbit,
  name: "Long Ear",
  eat() {         // [[HomeObject]] == longEar
    super.eat();
  }
};

*!*
longEar.eat();  // Long Ear eats.
*/!*
```

Every method remembers its object in the internal `[[HomeObject]]` property. Then `super` uses it to resolve the parent prototype.

`[[HomeObject]]` is defined for methods defined both in classes and in plain objects. But for objects, methods must be specified exactly the given way: as `method()`, not as `"method: function()"`.

In the example below a non-method syntax is used for comparison. `[[HomeObject]]` property is not set and the inheritance doesn't work:

```js run
let animal = {
  eat: function() { // should be the short syntax: eat() {...}
    // ...
  }
};

let rabbit = {
  __proto__: animal,
  eat: function() {
    super.eat();
  }
};

*!*
rabbit.eat();  // Error calling super (because there's no [[HomeObject]])
*/!*
```

## Static methods and inheritance

The `class` syntax supports inheritance for static properties too.

For instance:

```js run
class Animal {

  constructor(name, speed) {
    this.speed = speed;
    this.name = name;
  }

  run(speed = 0) {
    this.speed += speed;
    alert(`${this.name} runs with speed ${this.speed}.`);
  }

  static compare(animalA, animalB) {
    return animalA.speed - animalB.speed;
  }

}

// Inherit from Animal
class Rabbit extends Animal {
  hide() {
    alert(`${this.name} hides!`);
  }
}

let rabbits = [
  new Rabbit("White Rabbit", 10),
  new Rabbit("Black Rabbit", 5)
];

rabbits.sort(Rabbit.compare);

rabbits[0].run(); // Black Rabbit runs with speed 5.
```

Now we can call `Rabbit.compare` assuming that the inherited `Animal.compare` will be called.

How does it work? Again, using prototypes. As you might have already guessed, extends also gives `Rabbit` the `[[Prototype]]` reference to `Animal`.


![](animal-rabbit-static.png)

So, `Rabbit` function now inherits from `Animal` function. And `Animal` function normally has `[[Prototype]]` referencing `Function.prototype`, because it doesn't `extend` anything.

Here, let's check that:

```js run
class Animal {}
class Rabbit extends Animal {}

// for static propertites and methods
alert(Rabbit.__proto__ == Animal); // true

// and the next step is Function.prototype
alert(Animal.__proto__ == Function.prototype); // true

// that's in addition to the "normal" prototype chain for object methods
alert(Rabbit.prototype.__proto__ === Animal.prototype);
```

This way `Rabbit` has access to all static methods of `Animal`.

Please note that built-in classes don't have such static `[[Prototype]]` reference. For instance, `Object` has `Object.defineProperty`, `Object.keys` and so on, but `Array`, `Date` etc do not inherit them.

Here's the picture structure for `Date` and `Object`:

![](object-date-inheritance.png)

Note, there's no link between `Date` and `Object`. Both `Object` and `Date` exist independently. `Date.prototype` inherits from `Object.prototype`, but that's all.

Such difference exists for historical reasons: there was no thought about class syntax and inheriting static methods at the dawn of JavaScript language.

## Natives are extendable

Built-in classes like Array, Map and others are extendable also.

For instance, here `PowerArray` inherits from the native `Array`:

```js run
// add one more method to it (can do more)
class PowerArray extends Array {
  isEmpty() {
    return this.length == 0;
  }
}

let arr = new PowerArray(1, 2, 5, 10, 50);
alert(arr.isEmpty()); // false

let filteredArr = arr.filter(item => item >= 10);
alert(filteredArr); // 10, 50
alert(filteredArr.isEmpty()); // false
```

Please note one very interesting thing. Built-in methods like `filter`, `map` and others -- return new objects of exactly the inherited type. They rely on the `constructor` property to do so.

In the example above,
```js
arr.constructor === PowerArray
```

So when `arr.filter()` is called, it internally creates the new array of results exactly as `new PowerArray`. And we can keep using its methods further down the chain.

Even more, we can customize that behavior. The static getter `Symbol.species`, if exists, returns the constructor to use in such cases.

For example, here due to `Symbol.species` built-in methods like `map`, `filter` will return "normal" arrays:

```js run
class PowerArray extends Array {
  isEmpty() {
    return this.length == 0;
  }

*!*
  // built-in methods will use this as the constructor
  static get [Symbol.species]() {
    return Array;
  }
*/!*
}

let arr = new PowerArray(1, 2, 5, 10, 50);
alert(arr.isEmpty()); // false

// filter creates new array using arr.constructor[Symbol.species] as constructor
let filteredArr = arr.filter(item => item >= 10);

*!*
// filteredArr is not PowerArray, but Array
*/!*
alert(filteredArr.isEmpty()); // Error: filteredArr.isEmpty is not a function
```

We can use it in more advanced keys to strip extended functionality from resulting values if not needed. Or, maybe, to extend it even further.
