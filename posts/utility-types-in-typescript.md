```
author: @ekilah
created_at: 2022-01-27
updated_at: 2022-02-08
```


# [Utility types in TypeScript](#)

TypeScript (TS) offers a host of improvements over vanilla JavaScript with its compile-time type system. Codebases small and large can get a lot of benefits from the lightest applications of TS. The simplest, out-of-the-box features of TS can prevent whole classes of errors from occurring at run time and make collaborating way easier.

However, additions like that (e.g. telling TS which variables are expected to be `string` vs `number`) are just the tip of the iceberg. Experienced TS developers often seek out more advanced tools to squeeze every last drop out of the type system. By doing so, they can unlock additional layers of type safety by communicating more about the intent and meaning of their code to the TS compiler.

Utility types, our topic for today, are an essential tool in a TS developer's toolbox for doing just that. They give you a reusable way to define relationships between types and to operate on existing types to make new ones (often referred to as a type-level operation). Utility types are both fun to explore and fairly powerful, once you get familiar with them, so let's dive in!


## [Built-in utility types](#built-in-utility-types)

TypeScript actually ships with several utility types that are quite useful. These can be used as-is, and also as building blocks when making your own, more advanced utility types (more on that later). Here, we'll walk through a few of the built-in utility types with some example use cases.

### [`NonNullable`](#nonnullable)

Let's start with a simple one, `NonNullable<T>`, and break down how it works and how/why you might use it. Starting with the [definition](https://github.com/microsoft/TypeScript/blob/5142e37f2d2af3e1b4d071f82d58097f516cef1b/lib/lib.es5.d.ts#L1518-L1521):

```ts
/**
 * Exclude null and undefined from T
 */
type NonNullable<T> = T extends null | undefined ? never : T
```

As the documentation suggests, this utility type takes an input type (`T`), removes `null` and `undefined` from it (if present), and returns the result. `never` here is TypeScript's way of removing something from a type in these type-level ternary statements, called [Conditional Types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html).

Let's see an example. Given a type like `Foo`, we can programmatically create a related type `Bar`, where `Bar` is like `Foo` but doesn't include either of JavaScript's nullish types.

```ts
type Foo = string | boolean | null
type Bar = NonNullable<Foo> // Bar evaluates to `string | boolean`
```

Now, remember, this is a type-level operation; as with all TypeScript typings, this is a compile-time construct. `NonNullable` doesn't do anything to your data at runtime, it's just a way to change the type `Foo` into something else. It's still up to you to actually transform your data from one type to another.

Don't miss a big benefit to defining `Bar` in terms of `Foo` here: if we decide to add e.g. `number` to the list of types in `Foo` later, `Bar` will automatically also support `number` without human intervention. This kind of refactoring safety is a big benefit that utility types can provide you!



### [`Exclude` / `Extract`](#excludeextract)

`Exclude` and `Extract` help you pick and choose which members of a union type to keep or discard, similar to `NonNullable`. But, unlike `NonNullable`, `Exclude` and `Extract` let you choose exactly what to keep or discard, rather than assuming you want to discard `null` and `undefined` all the time.

You'll notice that the [definitions](https://github.com/microsoft/TypeScript/blob/5142e37f2d2af3e1b4d071f82d58097f516cef1b/lib/lib.es5.d.ts#L1513-L1516) of these two even look strikingly familiar:

```ts
/**
 * Exclude from T those types that are assignable to U
 */
type Exclude<T, U> = T extends U ? never : T

/**
 * Extract from T those types that are assignable to U
 */
type Extract<T, U> = T extends U ? T : never
```

In fact, `Exclude<Foo, null | undefined>` is equivalent to `NonNullable<Foo>`.

Let's walk through one example use case for `Exclude`. Say you have a function in your codebase that removes all the numbers from a heterogeneous list:

```js
function removeNumbers(list)
```

How can we nicely model this function with TypeScript? There are a few common, easy-way-out options:

```ts
function removeNumbers(list: any[]): any[]
function removeNumbers<T>(list: T): T[]
```

but surely we can do better. Neither of these tells TS that the return value's type should no longer include `number`. What we need is a way to relate the input type and the output type, but remove `number`. This is a great use case for `Exclude` and some generics:

```ts
function removeNumbers<T>(list: T[]): Exclude<T, number>[]
```

Now, if we have an input list of `number | string`, for example, TS will understand that the result of `removeNumbers` is a list of `string`, instead of seeing `number | string` still, or worse, `any`.


### [`Pick`/`Omit`](#pickomit)

Have you ever had a type or interface representing an object's structure in TypeScript, and wanted to replicate it, but with a few changes? `Pick` and `Omit` are perfect for that.

If you're thinking that this all sounds similar to what we already talked about with `Exclude` and `Extract`, great job paying close attention! `Pick` and `Omit` are to object types as `Extract` and `Exclude` are to union types; both let you keep or discard certain pieces of a given type. Remembering which is which can even be difficult sometimes! If anyone out there has a good memory device for this, let me know. 😅

#### [`Pick`](#pick)

A common use case for `Pick` comes up a lot for us in our React codebase here at Mercury. Let's say you have two components in a parent-child relationship, and several of the `props` passed to the child are also passed to the parent from some other component. For example, say all three of these props are passed to the parent, used there, and also passed to the child:

```ts
type SharedProps = {
  user: UserData
  business: BusinessData
  options: SomeOptions
}
```

The parent and child both have other props, so you can't reuse the same type for both. Since the components only share some props, it's tempting to copy & paste the shared parts to both components and move on. However, this can make things more difficult as your components and codebase grow:

* Refactoring is harder - if you change one of these shared props' type, you have to change it in many places
* Three shared props is probably fine, but it doesn't take long for this to grow to a much longer list of shared props
* You could export the `SharedProps` type above, but this pattern might get old: you have to define props in a shared location every time you have a parent/child relationship like this, give the shared type a good name, and pollute the global namespace with this extra export

The first two points can be summarized as `Don't Repeat Yourself`! We can use TS's built-in `Pick` to dry this code up nicely - here's the [definition](https://github.com/microsoft/TypeScript/blob/5142e37f2d2af3e1b4d071f82d58097f516cef1b/lib/lib.es5.d.ts#L1489-L1494):

```ts
/**
 * From T, pick a set of properties whose keys are in the union K
 */
type Pick<T, K extends keyof T> = {
  [P in K]: T[P]
}
```

Don't get too hung up on the syntax here. `Pick` takes an input (object) type, and, given a list of keys you want to keep from that object type, returns a type with only the matching key/value pairs from the input. It basically lets you `filter` an object type by key.

So for our example, we can reuse the child's props type in the parent's props type like so:

```ts
type ParentProps = Pick<ChildProps, 'user' | 'business' | 'options'> & {
  parentOnlyProp1: number
  parentOnlyProp2: string
  // ...
}
``` 

Now, if there's a fourth prop both components want to share later, you have fewer places to edit and keep in sync. It's also obvious to people reading this code later that there is a direct and intentional relationship between the props types of both components. 🎉

#### [`Omit`](#omit)

`Omit` is, roughly speaking, the exact opposite of `Pick`, in that it allows you to specify which keys to _leave out_ of the resulting type, rather than which to keep. In fact, the [definition](https://github.com/microsoft/TypeScript/blob/5142e37f2d2af3e1b4d071f82d58097f516cef1b/lib/lib.es5.d.ts#L1513-L1516) of `Omit` builds on `Pick`:

```ts
/**
 * Construct a type with the properties of T except for those in type K.
 */
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>
```

For example, say you have a type like this that represents data you read from a JSON API:

```ts
type UserDataFromAPI = {
  id: string
  name: string
  birthdate: string
  signedUpAt: string
  email: string
  // ...
}
```

Somewhere in your application, you decide to start rendering those dates (`birthdate` and `signedUpAt`), and get tired of calling `new Date()` every time you need a `Date` object from those date strings from the JSON API. So what you really want is this type instead:

```ts
type UserData = {
  id: string
  name: string
  birthdate: Date
  signedUpAt: Date
  email: string
  // ...
}
```

Now, you could make that type manually, which would work fine for a while. But, as soon as you start making edits to `UserDataFromAPI`, you'll realize that it's annoying to have to remember to manually update `UserData` as well. And as your team grows, you worry that other developers won't be aware of this tightly-linked relationship between `UserDataFromAPI` and `UserData`. `Omit` to the rescue!


With `Omit`, we can create `UserData` in such a way that it stays in sync with `UserDataFromAPI` over time, while still managing to make the edits we want to make to it. `Omit` takes two inputs, the object type to modify and the list of keys to remove from that type. We can then combine the output of `Omit` with an inline type that uses `Date` like we wanted:

```ts
type UserData = Omit<UserDataFromAPI, 'birthdate' | 'signedUpAt'> & {
  birthdate: Date
  signedUpAt: Date
}
```

> Note that `Exclude`, `Extract`, and `Omit` all suffer from a nuanced issue: they don't actually validate that the provided types/keys you specify for removal/extraction are actually members of the source type. For example, `Omit<UserDataFromAPI, 'doesNotExist'>` will compile just fine, and will be equivalent to `UserDataFromAPI`. There are fixes for this in a package called [`type-zoo`](https://github.com/pelotom/type-zoo) via `...Strict` varieties of these types (e.g. `OmitStrict`). These strict variants will throw compiler errors if you e.g. have a typo or miss something during a refactor, which is really useful!
> 
> `Pick` does not suffer from this problem, because it requires in `Pick<T, K extends keyof T>` that `K extends keyof T`.



  
### [`Required`](#required)

As mentioned above, `Exclude` and `Omit` are pretty similar in concept; the former is for union types and the latter is for object types. So is there something like `NonNullable`, which removes nullish types from a union, but for object types?

`Required` is pretty close to what you're looking for! Here's its [definition](https://github.com/microsoft/TypeScript/blob/5142e37f2d2af3e1b4d071f82d58097f516cef1b/lib/lib.es5.d.ts#L1475-L1480):

```ts
/**
 * Make all properties in T required
 */
type Required<T> = {
  [P in keyof T]-?: T[P]
}
```

As you might guess from this new syntax `-?`, `Required` removes any optionality (the `?` operator) from all keys of an object type.

So why did I say it is "pretty close" to what `NonNullable` does? Take this example type:

```ts
type Baz = {
  a?: number
  b: string | undefined
}
```

Which of these do you think `Required<Baz>` will be equivalent to? 


```ts
type This = {
  a: number
  b: string
}

// or

type That = {
  a: number
  b: string | undefined
}
```

It turns out that `Required` only removes the `?` from `a`, it does not actually change `b` at all, so `Required<Baz> = That`. Here's a [TS playground link](https://www.typescriptlang.org/play?#code/C4TwDgpgBAKhDOwoF4oG8BQVtQIYH4AuKAOwFcBbAIwgCcscrjFaBLEgcygB8oySAJhABm7CAIwBfDBgD0sqAAsA9gDc6UNRoAGAJQgBHMq1ri4ibVGDKo8CNADui3EmCLoyssDBeoAG2VlAGt4f1YgiABCDFBIKH0jEzMEJFQE41MBAB5zYAA+GXlSZWBWAGNoN2g6WmVaUOUSKAh1WhBNEg9hK3c7ZoAPCrAkWgAWABooOQUHaDLcJrtoKhLFPDxBKCooCjJELeh4SDLWUXFihwwyxv3aAEZidKSBXJR0aWuSW4AmR8MM5L7VBoXDEUbfD43EYAZj+iUyr2BTCgAHJhIEUZCviNRnCAS8Um8QWDvpNkWiMdJpltfLg-PAbCQSpVnEhtqxQohWH4-HgecpZgIrDYaKj+EJRJ0BCjJk5udBcFAOcVgNTGn52rQAKxKOjQMC4eB2UJVKDXChgeW0FGhMruMohK5QqDavHPRHoUFQcFk4jikRiARY24ANjdCMJwK9-sl4l9fEEAalwZGAHZw4DUp6-YnYwJ4xTlJiMEA) that demonstrates this in more detail, if you're curious.

> If `Baz`'s `a` was instead written as `a?: number | undefined`, you'd notice some likely-unexpected behavior from TypeScript's `Required`. See [this issue](https://github.com/microsoft/TypeScript/issues/31025#issuecomment-484734942) for more information if you want some extra credit!

`Required` can be useful when modeling optional options, say for a dollar value formatting function. Your user-facing type with optionality might look like this:

```ts
type OptionalOptions = {
  decimalPlaces?: number
  hideDollarSign?: boolean
}
```

Your formatting function's internals might need to define the set of defaults to use if some/all of the `OptionalOptions` aren't set by the user. `Required` is a great tool to use there:

```ts
const defaults: Required<OptionalOptions> = {
  decimalPlaces: 2,
  hideDollarSign: false,
}
```

Defining `defaults` this way will require that any additions/changes to `OptionalOptions` are also reflected in `defaults`, whether the dev making those changes knew about `defaults` or not. Leveraging the TS compiler to make sure related code is updated when a type changes is a huge win!


### [`Partial`](#partial)

[`Partial`](https://github.com/microsoft/TypeScript/blob/5142e37f2d2af3e1b4d071f82d58097f516cef1b/lib/lib.es5.d.ts#L1468-L1473) naturally follows `Required` - `Partial` will add optionality (via `?`) to every key of an object type for you.

```ts
/**
 * Make all properties in T optional
 */
type Partial<T> = {
  [P in keyof T]?: T[P]
}
```

If you'd like an example, think about how you could use `Partial` to derive `OptionalOptions` above, if you started with a type without the `?`s first.


## [...and many more!](#and-many-more)

We'll leave [the rest](https://github.com/microsoft/TypeScript/blob/5142e37f2d2af3e1b4d071f82d58097f516cef1b/lib/lib.es5.d.ts#L1468-L1561) for you to explore on your own. `Record`, `Parameters`, and `ReturnType` are some good ones to familiarize yourself with too, if you want a place to start!

## [Make your own](#make-your-own)

Using TS's built-in utility types, you can construct your own utility types that are either easier to reuse, have nicer names, or do more specific things you find useful. The main challenge, in my opinion, is mastering the relatively odd syntax TS uses to define them. Following the built-in types, and some great open-source resources, is my best recommendation for getting used to these!

Speaking of open-source resources, here are some links for you to explore, when you're ready for more advanced utility types:

- [type-zoo](https://github.com/pelotom/type-zoo)
	- Mostly simple utility types that could/should be included with TS itself
- [ts-toolbelt](https://github.com/millsp/ts-toolbelt)
	- Many more advanced utility types that build on what we covered here, and beyond
	- The author of `ts-toolbelt` also wrote a detailed [blog post](https://medium.com/free-code-camp/typescript-curry-ramda-types-f747e99744ab) on how he used utility types like these to model some of Ramda's more complicated functions
