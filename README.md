# A proposal for resource management in JS

*Initially inspired by [this ES Discuss thread](https://esdiscuss.org/topic/resource-management-eg-try-with-resources#content-2).*

JS really needs a resource management system. How often do you find yourself doing something like these?

```js
// Get some rows from a database, using the Promise disposer pattern
await db.connect(async connection => {
    const rows = await connection.query("SELECT * FROM TABLE")
    console.log(rows)
})

// Get some rows from a database, using `try`/`finally`
const connection = await db.connect()

try {
    const rows = await connection.query("SELECT * FROM TABLE")
    console.log(rows)
} finally {
    connection.close()
}

// Use a temp file
const tmp = require("tmp-promise")
const fs = require("fs/promises")

tmp.withFile(o => {
    console.log('Doing work with the temp file', o.path)
    await fs.writeFile(o.path, 'test', 'utf-8')
}).then(() => {
    console.log(`Temp file cleaned up!`)
})

// Multiple parallel connections
const Promise = require("bluebird")
await using([
    db.connect("db"),
    db.connect("db"), // Let's hope this doesn't fail... ;-)
], async ([conn1, conn2]) => {
    // use conn1 and conn 2 here
})
```

If so, it's pretty obvious this is all annoying boilerplate. What if, instead, it was like this?

```js
// Get some rows from a database
{
    with connection = await db.connect()
    const rows = await connection.query("SELECT * FROM TABLE")
    console.log(rows)
}

// Use a temp file
const tmp = require("tmp-promise")
const fs = require("fs/promises")

{
    with {path} = tmp.file()
    console.log('Doing work with the temp file', o.path)
    await fs.writeFile(o.path, 'test', 'utf-8')
}

console.log(`Temp file cleaned up!`)

// Multiple parallel connections
{
    with [conn1, conn2] = await Promise.withAll([
        () => db.connect(),
        () => db.connect(),
    ])
    // use conn1 and conn 2 here
}
```

## Proposal

So here's the basic context:

- A resource is an object with a `Symbol.dispose` method. Anything with that method can be used as a resource.

- `value[Symbol.dispose]()` - This is how you explicitly dispose of a resource. There are several ways a potential resource could implement this:
    - Built-in iterators and async iterators could implement this as an alias for `iter.return()`.
    - Subscriptions could implement this as an alias for `unsubscribe`.
    - Other more complex scenarios like async temp files would need this to destroy the file before returning.
    - It's obviously recommended that this be idempotent, although the spec makes no requirement of this.

- `value[Symbol.asyncDispose]()` - This is how you explicitly dispose of an async resource.
    - Built-in async iterators could implement this as an alias for `iter.return()`.
    - Node streams could use this as an alias for `.destroy()` or `.close()`, as appropriate.
    - Other more complex scenarios like async temp files would need this to destroy the file before returning.

- `with value = resource` - This is how you create a resource. It's closed at the end of the block it's in, so you don't need to care about that.
    - These are created as `const` variables, so they carry all the necessary restrictions with it. (This is to make it clearer what's being closed.)
    - If there's multiple `with` statements, they are closed in the reverse order they were created.
    - This doesn't actually conflict with the legacy `with` statement - patterns can't start with parentheses.
    - Resources are closed regardless of the completion type (it can be "normal", "throw", "return", or "break").
    - If an error occurs during closure, the first error to happen is propagated, but not until all resources within that block are closed (successfully or not).
    - In sync contexts, it only calls `value[Symbol.dispose]()`.
    - In async contexts, it calls `value[Symbol.asyncDispose]()` if the method exists, but it falls back to `value[Symbol.dispose]()` if it doesn't.
    - This gets the disposer method immediately and retains it until it needs to close the resource, so it can throw a `TypeError` at the creation site if it's missing either symbol.

- `Promise.withAll(factories: Iterable<() => Promise<asyncResource>>) -> Promise<[...values] & {[Symbol.asyncDispose](): void}>` - This creates and combines multiple async resources into an array of them, an array that also happens to have a `Symbol.asyncDispose` method set on it. If creating any resource fails, all remaining ones are closed as necessary, and the first error propagated. (The rest are swallowed.)
    - The main goal here is to open several resources in parallel and ensure they all close if one fails.
    - When `result[Symbol.asyncDispose]()` is called, this calls `value[Symbol.asyncDispose]()` for each resource if the method exists, but it falls back to `value[Symbol.dispose]()` if it doesn't. Also, all calls to `Symbol.asyncDispose` are awaited in parallel, like via `Promise.all`.
    - This gets the disposer method immediately for each resource and retains it until it needs to close it, so it can throw a `TypeError` immediately if it's missing either symbol.
    - This is analogous to Bluebird's [`Promise.using([...resources], ([...values]) => ...)`](http://bluebirdjs.com/docs/api/promise.using.html), so there's precedent. It's also hard to get right.
    - Maybe, a method to aggregate exceptions might be nice for this + `Promise.all`?

- `Promise.wrap(init: (body: (...values) => Promise<void>) => Promise<any>) -> Promise<[...values] & {[Symbol.asyncDispose](): Promise<void>}>`
    - This exists to adapt a promise disposer to an async resource, much like the `Promise` constructor is used to adapt callbacks to promises.
    - One could wrap an existing resource this way using an async function here with a `with` inside.
    - This is exposed as a built-in, since it's non-trivial to implement (you're literally having to do a restricted form of `call/cc` to do it).

Intentionally, there is no equivalent to `Promise.withAll` and `Promise.wrap` for sync resources. For `Promise.withAll`, the sync equivalent is just creating them with multiple `with` declarations - close-on-fail will just take care of itself naturally. For `Promise.wrap`, if you're dealing with a callback, JS doesn't have a "call with current continuation" primitive, so it's not possible to implement. If you're dealing with an object with an explicit disposer method, wrapping it is pretty trivial:

```js
function wrapFoo(...args) {
    const foo = getFoo(...args)
    return {value: foo, [Symbol.dispose]: () => foo.close()}
}

async function wrapFooAsync(...args) {
    const foo = await getFoo(...args)
    return {value: foo, [Symbol.asyncDispose]: async () => foo.close()}
}
```

## Why this way?

I know [this isn't the only proposal out there](https://github.com/rbuckton/proposal-using-statement), and many languages already use something like `using` or blocks:

- [Ruby uses blocks with `yield` + `ensure`](http://jakegoulding.com/blog/2012/10/01/using-ruby-blocks-to-ensure-resources-are-cleaned-up/)
- [Java uses `try`-with-resources](https://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html)
- [C# uses `using`](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/using-statement)
- [Python uses `with`](http://preshing.com/20110920/the-python-with-statement-by-example/)
- The promise disposer pattern is basically Ruby's `yield` + `ensure`, just done with promises and a higher-order function instead.

There's also the concept of "deferred" blocks:

- Swift has [`defer` blocks](https://www.hackingwithswift.com/new-syntax-swift-2-defer).
- Go has [`defer` statements](https://gobyexample.com/defer).

But here's why I chose this route:

- Most JS resources follow the concept of [RAII ("Resource Acquisition Is Initialization")](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization), both in the language (think: promises), in the DOM (think: recent constructor APIs), in Node (think: file handles), and in userspace (think: [`tmp`](https://www.npmjs.com/package/tmp)).
    - [C++](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization#C++11_example) and [Rust](https://doc.rust-lang.org/rust-by-example/scope/raii.html) both use and *strongly* encourage RAII. Their resource management style is what this proposal was inspired by, and since they have to deal with resources and potential leaks heavily, and so they've had to sugar over them to make them easier to use. I mean, take a look at [their example for files](https://rustbyexample.com/std_misc/file/open.html) - it's almost like magic, but in a good way. C++ works pretty similarly for sockets and other file management - it's basically effortless.
    - Resources should be easy to deal with, hence the simple replacement of a token.

- You don't have to deal with the pyramid of doom when opening resources, even when they depend on other resources.
    - Unlike the [`using` proposal](https://github.com/rbuckton/proposal-using-statement), you can even drop a `console.log` between opening two different resources.
    - And it's *pretty obvious* that you could run into this issue in a hurry with Promises unless you're lucky enough to have a very non-trivial utility that can turn a `Iterable<(body: (value: T) => Promise<U>) => Promise<U>>` into a `(body: (value: [...T]) => Promise<U>) => Promise<U>` while handling all the edge cases that can arise from that (like timings).

- Reusing an existing keyword is always nice, especially when it still remains meaningful. Python's use of `with` was another influence, and it made me feel it was a little more justified.

- Block scopes are nice for restricting scopes already. Why invent yet another concept, when we can just reuse what we have?

## Polyfills?

I don't have any polyfills ready yet, but I would like to eventually. This also includes syntax, so it'd also be pending a Babel parser plugin.
