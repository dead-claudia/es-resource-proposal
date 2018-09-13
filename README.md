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
await Promise.using([
    db.connect("db1"),
    db.connect("db2"), // Let's hope this doesn't fail... ;-)
], async ([conn1, conn2]) => {
    // use conn1 and conn 2 here
})

// Guarding a critical section
const lock = new Lock()
// Later on...
lock.acquire()
try {
    // Do something with a SAB.
} finally {
    lock.release()
}
```

If so, it's pretty obvious this is all annoying boilerplate. What if, instead, it was like this?

```js
// Get some rows from a database
using (const connection = db.connect()) {
    const rows = await connection.query("SELECT * FROM TABLE")
    console.log(rows)
}

// Use a temp file
const tmp = require("tmp-promise")
const fs = require("fs/promises")

using (const {path} = tmp.file()) {
    console.log('Doing work with the temp file', o.path)
    await fs.writeFile(o.path, 'test', 'utf-8')
}

console.log(`Temp file cleaned up!`)

// Multiple parallel connections
using (const [conn1, conn2] = Promise.usingAll([
    () => db.connect("db1"),
    () => db.connect("db2"),
])) {
    // use conn1 and conn 2 here
}

// Guarding a critical section
const lock = new Lock()
// Later on...
using (lock) {
    // Do something with a SAB.
}
```

## Proposal

So here's the basic idea:

### Resources

A resource is an object with a `res[Symbol.use]()` method returning either `null`/`undefined` (non-fatal error) or a `{value, close(): any}` pair.

- `using` blocks invoke `Symbol.use` to get a handle, use `.value` to get the raw binding to expose, then invoke `.close()` to close it (awaiting in `async` functions and `Promise.usingAll`).
- `Symbol.use` methods can also choose to return `null`/`undefined`, if the resource isn't usable for some reason, but it's minor, obvious what likely happened, and easily recoverable (like failing to acquire a lock).

There are several ways a resource could implement `Symbol.use`. Here's a list of several examples, some for language built-ins, some for proposed built-ins, and some for Node built-ins:

```js
// Example built-in use with array buffers
ArrayBuffer.prototype[Symbol.use] =
SharedArrayBuffer.prototype[Symbol.use] = function () {
    return {value: this, close: async () => {
        const buffer = this.[[ArrayBufferData]]
        %DetachArrayBuffer(this)
        await gcCollect(buffer) // not a real operation, just recommended
    }}
}

// Example built-in use with observables
Observable.prototype[Symbol.use] = function () {
    const subs = new Set()
    return {
        value: new Observable(observer => this.subscribe({
            sub: undefined,
            start(sub) { subs.add(this.sub = sub) },
            next(v) { observer.next(v) },
            error(v) { subs.remove(this.sub); observer.error(v) },
            complete() { subs.remove(this.sub); observer.complete() },
        })),
        close: () => {
            for (const sub of subs) sub.unsubscribe()
        }
    }
}

// Example library use with atomic locks (mutexes)
class Lock {
    constructor(buf = new SharedArrayBuffer(1)) { /* ... */ }
    get buffer() { /* ... */ }
    acquire() { /* ... */ }
    release() { /* ... */ }
    [Symbol.use]() {
        return this.acquire()
            ? {close: () => this.release()}
            : undefined
    }
}

// Example Node use with streams
const {Readable, Writable} = require("stream")

Readable.prototype[Symbol.use] =
Writable.prototype[Symbol.use] = function () {
    return {value: this, close: () => this.destroy()}
}

// Example Node use with child processes
const {ChildProcess} = require("child_process")
const throwError = err => { throw err }

ChildProcess.prototype[Symbol.use] = function () {
    // Private method used here for more sane resource management.
    const hasRef = this._handle.hasRef()
    if (!hasRef) this.ref()
    return {value: this, close: () => {
        if (!hasRef) this.unref()
        this.prependOnceListener("error", throwError)
        try {
            this.kill("SIGTERM")
        } finally {
            this.removeListener("error", throwError)
        }
    }}
}
```

### `using` blocks

These are the standard way of consuming resources.

```js
// Normal form
using (let value = resource) {
    // ...
}

// Multiple resources are allowed
using (let value1 = resource1, value2 = resource2) {
    // ...
}

// Invoke `else` if `resource` is `null`/`undefined`
using (let value = resource) {
    // ...
} else {
    // ...
}
```

This is how you consume a resource. It's closed at the end of the block it's in, so you don't need to care about that.

- These are created using standard declarations, so they carry all the necessary restrictions with it.
- If an error occurs during closure, the first error to happen is propagated, but not until all resources within that block are closed (successfully or not).
- `using (let foo = one, bar = two) { ... }` is equivalent to `using (let foo = one) using (let bar = two) { ... }`
- It calls `let handle = resource[Symbol.use]()`, sets the binding to `handle.value` while destructuring as appropriate, and then invokes `handle.close()` on the returned handle after the block.
- If any resource or corresponding handle is `null`/`undefined`, it invokes the `else` block instead. This is to support things like `using (let res = getResource()) { ... } else { return false }` or similar, where a minor failure is easily tolerated and obvious from context.
    - If other resources were successfully created, they're closed *before* invoking the `else` block.
- The `else` block is optional, but if you omit it, it defaults to throwing a `TypeError` instead of doing nothing, so obvious errors don't get swallowed.

### Anonymous resource consumption

Sometimes, you want to consume a resource, but it doesn't make sense to give the result a name - locks are a good example of this.

```js
// Normal form
using (resource) {
    // ...
}

// Multiple resources are allowed
using (resource1, resource2) {
    // ...
}

// Invoke `else` if `resource` is `null`/`undefined`
using (resource) {
    // ...
} else {
    // ...
}
```

This lets you consume a resource anonymously.

- This form is identical to the previous form, except no binding is generated and `handle.value` is never accessed. Similarly, the `else` block is optional, defaulting to throwing a `TypeError`.
- `using (one, two) { ... }` is equivalent to `using (one) using (two) { ... }`, just like with the binding form.

Note that this form can't be mixed with the declaring form - `using (one, let bar = two)` and `using (let foo = one, two)` are both invalid.

- In the first case, it's pretty obvious what the correct interpretation is.
- In the second, it's unclear whether `two` is being declared as `undefined` or read as a resource.
- If the second naturally falls out from the first, yet is incredibly ambiguous, the first is probably also a bad idea.

### Utility method: `Promise.usingAll()`

Just like how you often want to perform and await multiple operations in parallel using `Promise.all`, you may also wish to create and use multiple resources in parallel.

```js
using (const [value1, value2] = await Promise.usingAll([
    () => resource1,
    () => Promise.resolve(resource2),
])) {
    // ...
}
```

This creates and initializes multiple async resources in parallel, merging their values into a single array.

- If any resource throws/rejects when initializing, all the remaining ones are closed upon resolution, and the first error is propagated via an initial rejection.
- If any resource returns/resolves with `null`/`undefined` (or its `resource[Symbol.use]()` method returns with `null`/`undefined`), all the remaining ones are closed upon resolution, and it resolves with `undefined` or rejects with the first error encountered.
- While closing, if any resource fails to close, the first error encountered is propagated, and the rest are silently swallowed.
- The main goal here is to open several resources in parallel and ensure they all close in parallel if any one fails for any reason.
- This is better than multiple `using` declarations, since those are executed sequentially.
- This is analogous to Bluebird's [`Promise.using([...resources], ([...values]) => ...)`](http://bluebirdjs.com/docs/api/promise.using.html), so there's precedent. It's also non-obvious to get right due to all the edge cases.
- Maybe, a method to aggregate exceptions might be nice for this + `Promise.all`?

There is intentionally no equivalent for sync resources, since it'd be generally equivalent to just a sync `using` statement. And if you wanted to manage them explicitly, it's not nearly as fraught with confusing edge cases as async resources are.

### Utility method: `Atomics.lock()`

With the ability to auto-close things, you could much more easily create mutexes and locks, which would make it much easier to deal with shared array buffers and other similar stuff at a higher level.

```js
const lock = Atomics.lock(buffer=new SharedArrayBuffer(1))
// Later on...
using (lock) {
    // Do something with a few SABs.
}
```

This creates a recursive lock for you to use, attached to a shared array buffer you can use to mirror it across worker boundaries. It would return a resource for guarding critical sections with `using`.

This would have only one extra method, `lock.withTimeout(ms)`, but would otherwise just have the bare minimum:

- `let proxy = lock.withTimeout(ms)` - Create a lock proxy resource that carries a finite timeout of `ms` milliseconds.
- `let handle = lock[Symbol.use]()` - Acquire the lock and return the resulting handle.
- `let handle = proxy[Symbol.use]()` - Acquire the lock, returning the resulting handle, or return `undefined` if the timeout is reached.
- `handle.close()` - Release the lock.
- There is no `handle.value`, because it's redundant and useless.

An implementation might choose to elide the intermediate allocation if `Symbol.use` is unmodified. In JIT code, this could be compiled down to a standard lock sequence updating the object directly, near-zero overhead in optimized code.

## Inspiration

I know [this isn't the only proposal out there](https://github.com/rbuckton/proposal-using-statement), and many languages already use something like `using` or blocks:

- [Ruby uses blocks with `yield` + `ensure`](http://jakegoulding.com/blog/2012/10/01/using-ruby-blocks-to-ensure-resources-are-cleaned-up/)
- [Java uses `try`-with-resources](https://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html)
- [C# uses `using`](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/using-statement)
- [Python uses `with`](http://preshing.com/20110920/the-python-with-statement-by-example/)
- The promise disposer pattern is basically Ruby's `yield` + `ensure`, just done with promises and a higher-order function instead.

There's also the concept of "deferred" blocks:

- Swift has [`defer` blocks](https://www.hackingwithswift.com/new-syntax-swift-2-defer).
- Go has [`defer` statements](https://gobyexample.com/defer).

And of course, there's [RAII ("Resource Acquisition Is Initialization")](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization), used by both [C++](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization#C++11_example) and [Rust](https://doc.rust-lang.org/rust-by-example/scope/raii.html).

But I chose to go a slightly different route and separate acquisition and release instead, operating on handles rather than the values directly. [I've already found that you can abuse generators and `for ... of` for this](https://esdiscuss.org/topic/resource-management), but it doesn't exactly provide a proper resource management solution, and the little it *does* provide doesn't quite account for all the edge cases you'd need to address.

- In Java and similar, it's *very* common to close a resource in the last part of a `try ... finally` statement. This got *so* common that Java's `try`-with-resources and C#'s `using` were both added to sugar over this very idiom. Here's an example [derived from their tutorial documentation](https://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html):

    ```java
    // Java 6
    static String readFirstLineFromFile(String path) throws IOException {
        BufferedReader br = new BufferedReader(new FileReader(path));
        try {
            return br.readLine();
        } finally {
            if (br != null) br.close();
        }
    }

    // Java 7
    static String readFirstLineFromFile(String path) throws IOException {
        try (BufferedReader br = new BufferedReader(new FileReader(path))) {
            return br.readLine();
        }
    }
    ```

- You can make an easy distinction between a common soft, non-fatal failure like a lock timeout or attempting to retrieve a non-existent object, and a potentially fatal one like a missing file. It works almost like an option of sorts, where you have options beyond a simple `try`/`catch` to handle issues that come up. It's also easier to avoid the trap of catching errors that aren't really errors, as `catch { ... }` would be pretty tempting for a lot of cases, even though if you're dealing with I/O, you should at least be *aware* things could fail.

- It's common in Ruby to implement resources via something like this:

    ```rb
    def open(*args)
        res = open_handle(*args)
        yield res
    ensure
        res.close!
    end
    ```

    In fact, even the language's built-in `File.open` works this way:

    ```rb
    File.open("path/to/file.txt") do |fd|
        # do things
    end
    ```

- It's idiomatic in RAII situations to have separate acquire/release steps, and that decoupling makes it more flexible while still retaining *some* memory safety. With this proposal, you can have things like `using (const object = somePool) { ... }` and it just works. By also separating the acquire/release steps, it also prevents certain very obviously absurd things, like ["releasing" locks that weren't acquired in the first place](https://www.youtube.com/watch?v=IcgmSRJHu_8).

## Polyfills?

I don't have any polyfills ready yet, but I would like to eventually. This also includes syntax, so it'd also be pending a Babel parser plugin.
