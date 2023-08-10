---
title: "Implementing Elixir-like protocols in TypeScript using Proxies"
seoDescription: "Investigate Elixir-style protocols in TypeScript with Proxies for adaptable polymorphism, promoting strong, expressive code and reusability"
datePublished: Thu Aug 10 2023 12:38:13 GMT+0000 (Coordinated Universal Time)
cuid: cll55ahuj000f09meh8zy3xg6
slug: implementing-elixir-like-protocols-in-typescript-using-proxies
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/ZH4FUYiaczY/upload/5ea7c94dcbc27d6481c62e8e3fc8b659.jpeg
tags: proxy, elixir, typescript, functional-programming, protocols

---

## Introduction

In this part of experimenting with proxies, we'll explore an alternative approach to polymorphism by leveraging the power of proxies available in JavaScript and TypeScript. Our main goal is to compose a solution similar to Elixir Protocols, which provide a clean and flexible way to define and implement polymorphic behavior across different data types.

### What is Elixir?

Elixir, a functional programming language built on the Erlang VM, offers a unique approach to polymorphism through protocols. These protocols allow developers to define a set of functions that can be implemented by various data types, promoting code reusability and extensibility. By combining TypeScript's strong typing with JavaScript's dynamic nature, we can create a powerful and expressive solution that mimics Elixir-like protocols.

## What are Elixir Protocols?

Elixir protocols are a powerful language feature that enables polymorphism, allowing developers to define a set of functions that can be implemented by different data types. This promotes code reusability, extensibility, and a clean separation of concerns. Let's dive deeper into the concept of Elixir protocols and understand how they work.

### Protocol Definition

For our example, we'll simplify what protocols are by using Alchemist Camps' gentle introduction.

%[https://www.youtube.com/watch?v=NB58aFTZmrE] 

In Elixir, a protocol is defined using the `defprotocol` keyword, followed by the protocol name and a block containing function signatures. These function signatures define the expected behavior for any data type that implements the protocol. Here's an example of a simple protocol definition:

```elixir
defprotocol Animal do
  @fallback_to_any true
  def greet(arg)
  def speak(arg)
  def warn(arg)
  defdelegate kind(arg), to: Animal.Any
  defdelegate describe(arg), to: Animal.Any
end
```

In this example, we define a `Animal` protocol with three functions `greet/1`, `speak/1`, and `warn/1` which takes a single argument `arg`. The `@fallback_to_any` attribute helps us describe what default speech an animal should say when there is no particular implementation made for it. We can implement defaults like so:

```elixir
defimpl Animal, for: Any do
  def greet(_), do: ""
  def speak(_), do: ""
  def warn(_), do: ""

  def kind(animal) do
    animal.__struct__
  end

  def describe(animal) do
    IO.puts("""
    This animal is a #{Animal.kind(animal)} named #{animal.name}.
    It says "#{Animal.warn(animal)}" when it's scared.
    It says "#{Animal.speak(animal)}" to communicate.
    It says "#{Animal.greet(animal)}" when its friends arrive.
    """)
  end
end
```

### Protocol Implementation

Like the example above, once a protocol is defined, various data types can implement the protocol by providing concrete implementations for the defined functions. This is done using the `defimpl` keyword, followed by the protocol name, the `for` keyword, the data type, and a block containing the function implementations. Here's an example of implementing the `Animal` protocol for the `Dog` data type:

```elixir
defmodule Dog do
  @enforce_keys [:name]
  alias __MODULE__
  defstruct name: ""

  def new(name) do
    %Dog{name: name}
  end
end

defimpl Animal, for: Dog do
  def greet(_), do: "woof woof!"
  def speak(_), do: "woof!"
  def warn(_), do: "growl"
end
```

In this example, we provide an implementation of the `greet/1`, `speak/1`, and `warn/1` functions for the `Dog` data type, create speeches as a string representation.

We can similarly make one for a `Cat` data type:

```elixir
defmodule Cat do
  @enforce_keys [:name]
  alias __MODULE__
  defstruct name: ""

  def new(name) do
    %Cat{name: name}
  end
end

defimpl Animal, for: Cat do
  def greet(_), do: "..."
  def speak(_), do: "meow"
  def warn(_), do: "hiss!"
end
```

### Protocol Dispatch

Elixir protocols use dynamic dispatch to determine the appropriate implementation for a given data type at runtime. When a protocol function is called with a specific data type, Elixir looks up the corresponding implementation and invokes the function. If no implementation is found, a `Animal.UndefinedError` is raised.

Here's an example of using the `Animal` protocol to express speech from different data types to their string representations:

```elixir
iex> buster = Dog.new("Buster")
%Dog{name: "Buster"}
iex> buster |> Animal.describe
"""
This animal is a Dog named Buster.
It says "growl" when it's scared.
It says "woof!" to communicate.
It says "woof woof!" when its friends arrive.
"""
```

In this example, the `` `Animal.describe/1` `` function is piped from our `Dog` object and will run the `` `Animal.kind/1` `` `` `Animal.greet/1` ``, `` `Animal.speak/1` ``, `` `Animal.warn/1` `` functions to return our string. We can do the same thing for our `Cat` constructor:

```elixir
iex> tabby = Cat.new("Tabby")
%Cat{name: "Tabby"}
iex> tabby |> Animal.describe
"""
This animal is a Cat named Tabby.
It says "hiss" when it's scared.
It says "meow" to communicate.
It says "..." when its friends arrive.
"""
```

The same happens for `Cat`, but with different results, based on our implementation. Elixir dynamically dispatches the function call to the appropriate implementation for each data type.

## Is this possible in TypeScript

Yes, this kind of polymorphism is completely possible in TypeScript with the power of proxies. We essentially need:

* One protocol dispatcher and one protocol implementer
    
* A way to determine algebraic data types (ADTs); for our case, we'll use an object that defines a `type` property
    
* A way to differentiate between ADTs and native JavaScript data types
    
* A way to arbitrarily define methods by a type (here is where our proxies come in)
    

Below is an implementation that does all of that. There's a lot going on that deserves its own article, but the idea will become clear towards the end of this post.

```typescript
type ConstructorName<T> = T extends Object ? T['constructor']['name'] : keyof any
function typeOf<T>(value: T) {
  return Object.prototype.toString.call(value).slice(8, -1);
}

function createProtocol<T, P>(): [
  Required<P>,
  { __protocol__?: Partial<P> }
  & { Any?: P }
  & { [K in DataType<T>]: P }
  & { [K in ConstructorName<T>]: P }
]

function createProtocol(): any {
  const implementDict = {} as any;

  const protocol = new Proxy(
    (property: any) => (dataType: any, ...rest: any[]) => {
      let type = '';
      switch (typeOf(dataType)) {
        case 'Object': {
          if (dataType?.constructor?.name !== 'Object') type = dataType.constructor.name;
          if (!dataType?.type) type = '$Object';
          if (typeof dataType.type !== 'string') type = '$Object';
          else type = dataType?.type;
          break;
        }
        default: type = `$${typeOf(dataType)}`; break;
      }
      return implementDict?.[type]?.[property]?.(dataType, ...rest)
        ?? implementDict?._Protocol?.[property]?.(dataType, ...rest)
        ?? implementDict?._Any?.[property]?.(dataType, ...rest);
    },
    { get: (target, prop) => target(prop) },
  );

  const implement = new Proxy(implementDict, {
    set(target, prop, value) {
      if (target?._Any) {
        target[prop] = { ...target._Any, ...value };
      }
      target[prop] = value;
      return true;
    },
  });

  return [protocol, implement];
}
```

### Recreating the `Animal` Protocol

We'll use our `createDataType` that we made from a previous post: [A neat way to write algebraic data types in TypeScript using Proxies](https://monotykamary.hashnode.dev/a-neat-way-to-write-algebraic-data-types-in-typescript-using-proxies). We will use it to simplify composing our data types and create our `Dog` and `Cat` constructors and implement our `Animal` protocol similar to the above examples:

```typescript
// compose our Dog and Cat constructors
type AnimalType =
  | { type: 'Dog', name: string }
  | { type: 'Cat', name: string }

const { Cat, Dog } = createDataType<AnimalType>();

// create our protocol and define it's types
const [Animal, implementAnimal] = createProtocol<AnimalType, {
  greet(arg: AnimalType): string
  speak(arg: AnimalType): string
  warn(arg: AnimalType): string
  kind?(arg: AnimalType): string
  describe?(arg: AnimalType): string
}>();

// equivalent to our "@fallback_to_any true"
implementAnimal._Any = {
  greet: () => '',
  speak: () => '',
  warn: () => '',
  kind: ({ type }) => type,
  describe: (animal) => `
    This animal is a ${Animal.kind(animal)} named ${animal.name}.
    It says "${Animal.warn(animal)}" when it's scared.
    It says "${Animal.speak(animal)}" to communicate.
    It says "${Animal.greet(animal)}" when its friends arrive.
  `,
};

// implement our Dog type
implementAnimal.Dog = {
  greet: () => 'woof woof!',
  speak: () => 'woof!',
  warn: () => 'growl',
};

// implement our Cat type
implementAnimal.Cat = {
  greet: () => '...',
  speak: () => 'meow',
  warn: () => 'hiss',
};
```

We can then dispatch the `Animal.describe` function like in Elixir, and we should expect the same result.

```typescript
const buster = Dog({ name: 'Buster' });
console.log(Animal.describe(buster));
/*
This animal is a Dog named Buster.
It says "growl" when it's scared.
It says "woof!" to communicate.
It says "woof woof!" when its friends arrive.
*/

const tabby = Cat({ name: 'Tabby' });
console.log(Animal.describe(tabby));
/*
This animal is a Cat named Tabby.
It says "hiss" when it's scared.
It says "meow" to communicate.
It says "..." when its friends arrive.
*/
```

We have essentially created something extremely similar to Elixir's protocol. This simple example might not be enough to give us an idea of how powerful protocols are as a method for polymorphism. We have to go big.

### Mimicking the `Enumerable` Protocol

#### What is the `Enumerable` Protocol?

The Enumerable Protocol is a core feature in Elixir that allows data structures to be iterated over and processed using a set of algorithms provided by the Enum and Stream modules. By implementing the Enumerable Protocol, a data type can be used with various functions from the Enum module, such as `map`, `filter`, `reduce`, and many others.

To implement the Enumerable Protocol, a data type must provide implementations for four functions: `reduce/3`, `count/1`, `member?/2`, and `slice/1` . The core of the protocol is the `reduce/3` function, which is responsible for performing the reducing operation on the enumerable data structure.

#### A simple map example

On top of the four functions we'll define as mandatory for the implementation, we'll simplify our `Enumerable` example by allowing the protocol to define a function `map`, very similar to JavaScript's array's `map`.

```typescript
// define a common `Enumerable` type so that we can reference the type inside our implemented functions
type Enumerable = any[]
const [Enum, implementEnum] = createProtocol<Enumerable, {
  // we'll make it mandatory to implement the four required functions
  reduce<A>(value: Enumerable, acc: A, fn: (item: any, acc: A) => A): A
  count(value: Enumerable): number
  member(value: Enumerable, element: any): boolean
  slice(value: Enumerable, start: number, end: number): Enumerable
  // on top of our four functions, we'll use the implemented functions to compose our `map` function
  map?<E extends Enumerable, V>(enumerable: E, fn: (item: Item<E>) => V): V[]
}>();
```

Let's first define how our `map` function will be composed. We will use `implementEnum._Protocol` to implement our dispatch functions based on the four functions each data type is required to implement:

```typescript
implementEnum._Protocol = {
  map: (enumerable, fn) => {
    return Enum.reduce(enumerable, [] as any[], (item, acc) => {
      const value = fn(item);
      acc.push(value);
      return acc;
    });
  },
}
```

*For this case, we'll assume any output from other functions in the protocol will return an* ***array***.

Let's implement the `Enumerable` protocol on JavaScript's native `Array` type. We will use the `$` symbol to differentiate between object-defined data types and native data types:

```typescript
implementEnum.$Array = {
  reduce: (array: any[], acc, fn) => array.reduce((acc1, item) => fn(item, acc1), acc),
  count: (array: any[]) => array.length,
  member: (array: any[], element) => array.some((item) => item === element),
  slice: (array: any[], start, end) => array.slice(start, end),
};
```

Then, from our newly implemented native `Array` type, we now have the `map` function for free. We can call it just like the `map` method in normal arrays:

```typescript
implementEnum.$Array = {
  reduce: (array: any[], acc, fn) => array.reduce((acc1, item) => fn(item, acc1), acc),
  count: (array: any[]) => array.length,
  member: (array: any[], element) => array.some((item) => item === element),
  slice: (array: any[], start, end) => array.slice(start, end),
};
```

```typescript
const array = [0, 1, 2, 3, 4, 5, 6];
const arrayTimes2 = Enum.map(array, (a) => a * 2);

console.log(arrayTimes2); // [0, 2, 4, 6, 8, 10, 12]
```

#### Now with an ADT: a singly linked-list

Doing this with native data types is a bit boring and unneeded. The real magic, however, happens when we apply it to our custom data types. Let's implement it on a linked-list example.

1. First, we'll construct our linked-list:
    

```typescript
type ListType =
  | { type: 'Cons', head: number, tail: ListType }
  | { type: 'Nil' }

const List = createDataType<ListType>();
const Cons = (head: number, tail: ListType) => List.Cons({ head, tail });
const Nil = List.Nil();
```

1. Then we'll define our four functions for our `ListType`. As from our `map` dispatch function, the secret sauce lies in our `reduce` implementation. Each of our implemented function will use recursion to do BFS traversal for our linked-list:
    

```typescript
// update our Enumerable type to include our ListType data type
type Enumerable = ListType | any[]

implementEnum.Cons = {
  reduce: (list: ListType, acc, fn) => {
    const traverse = (list: ListType) => {
      switch (list.type) {
        case 'Cons':
          fn(list.head, acc);
          traverse(list.tail);
        case 'Nil':
          return acc;
      }
    }
    return traverse(list);
  },
  count: (list: ListType) => {
    let acc = 0
    const traverse = (list: ListType): number => {
      switch (list.type) {
        case 'Cons':
          acc += 1;
          traverse(list.tail);
        case 'Nil':
          return acc;
      }
    }
    return traverse(list);
  },
  member: (list: ListType, element: number): boolean => {
    const traverse = (list: ListType) => {
      switch (list.type) {
        case 'Cons':
          if (list.head === element) {
            return true
          }
          traverse(list.tail);
        case 'Nil':
          return false;
      }
    }
    return traverse(list);
  },
  slice: (list: ListType, start, end) => {
    let index = -1

    const traverse = (list: ListType): ListType => {
      index += 1
      switch (list.type) {
        case 'Cons':
          if (index == start) {
            return Cons(list.head, traverse(list.tail))
          }
        case 'Nil':
          return Nil
      }
    }
    return traverse(list);
  },
};
```

1. Now we can run the same `map` function on our linked-list and see the same result, with our linked-list converted into an enumerable array:
    

```typescript
const list = Cons(1, Cons(2, Cons(3, Cons(4, Cons(5, Cons(6, Nil))))));
const listTimes2 = Enum.map(list, (l) => l * 2);

console.log(listTimes2); // [0, 2, 4, 6, 8, 10, 12]
```

## Final Thoughts

Although typing could be improved a lot with more granular typing patterns, we now have a new way to do polymorphism in TypeScript with pattern matching. With our implementation, we allow a somewhat clean and flexible way to define and implement polymorphic behavior across different data types. The solution, feature-wise, is exactly like Elixir Protocols and works as expected for our simple `Animal` Protocol, as well as our more advanced `Enumerable` Protocol, especially on our custom data type.