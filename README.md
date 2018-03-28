# A proposal for resource management in JS

⚠️⚠️⚠️ This proposal is under construction. ⚠️⚠️⚠️

Things may look incomplet and incorrekt, and probably looks like a giant stage -1000 strawman. Please proceed at your own risk, as things might look completely and totally confusing to you. 😉

- A resource is an object with a `Symbol.close` method.

- `value[Symbol.close]()` - Close the resource.
    - It *should* be reentrant as well as idempotent, but the spec makes no requirement of this.
    - Iterators *should* implement this as `iter.return()`, dropping the return value
    - Async iterators *should* implement this as `iter.return()`, dropping the returned promise's resolution value.
    - The Observable proposal's `Subscription` should add this as an alias of `unsubscribe` and accept objects with this as an alias of `unsubscribe` for subscription-like objects
    - Note: this should *not* be used to replace cancellation (like aborting an XHR) - only things that can conceptually be closed (like a stream or coroutine).

- `with value = resource` - Allocate a sync resource
    - `value` is actually a pattern. Any assignment pattern works.
    - It does *not* conflict with the legacy `with` statement, because patterns can't start with a left parenthesis.
    - At the end of the block, each resource's `Symbol.close` method is invoked from last to first.
    - In async contexts, all implicit `Symbol.close` method calls are awaited in parallel.

- Previously created resources are closed if later resources throw on creation
    - This just falls out of the lexical computation order

- `Resource.wrap(init: (body: (value) => Promise<void>) => Promise<void>) -> Promise<AsyncResource & {value: T}>`
    - Adapts an existing promise disposer callback to a resource.
    - Similar to Promises with callback APIs.
    - Sometimes, it's easier to use this method rather than specifying a `Symbol.close` method.
    - Note: `body` is guaranteed never to throw, even if `init` throws.

- `Resource.join(iter)` is like `Promise.all`, but only accepts an `Array<() => AsyncResource>`.
    - Thunks are required to detect sync errors when initializing async resources, to close those successfully created.
    - Returns a promise to an array of resources with a `Symbol.close` method added to it.

- `with` declarations create `const` bindings (to make it clear you can't change what's being closed)

- It's block scoped, so it's easier to use multiple resources at once, even when they're dependent.

- It's a lot simpler to compose multiple close methods than it is nested callbacks.
    - Consider the `Resource.join` implementation below
    - And trying to execute such callbacks in parallel isn't exactly trivial, as one of the helpers below shows

- It's easily optimized and trivially desugared (even by transpilers).

- It's simple to understand and easy to explain.

- It's opinionated in favor of [RAII ("Resource Acquisition Is Initialization")](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization), but most JS APIs already follow that to some extent.
    - Consider: promises, array buffers, DOM's newer APIs, Node's external APIs, etc.

- It's opinionated in favor of FIFO resource management, but this is already mandated by the promise disposer pattern.

- It avoids all the nesting issues and other functional confusion frequent with promise disposer patterns:

    1. Simple creating a connection to use.

        ```js
        // Promise disposer pattern
        const rows = await getConnection(async connection => {
            return connection.queryAsync("SELECT * FROM TABLE")
        })

        console.log(rows);

        // This proposal
        let rows = (async () => {
            with connection = await getConnection()
            return connection.queryAsync("SELECT * FROM TABLE");
        })()

        console.log(rows)
        ```

    2. Using a temp file

        ```js
        // Common stuff
        const tmp = require("tmp");
        const fs = require("fs");
        const util = require("util")

        const exists = util.promisify(fs.exists)
        const writeFile = util.promisify(fs.writeFile)

        // Promise disposer pattern
        async function withTempFile(opts, func) {
            if (func == null) { func = opts; opts = undefined }
            const [path, fd, cleanupCallback] = await new Promise((resolve, reject) => {
                tmp.file(opts, (err, ...ret) =>
                    err != null ? reject(err) : resolve(ret))
            })

            try {
                return await func(path, fd)
            } finally {
                cleanupCallback()
            }
        }

        ;(async () => {
            let rememberPath

            try {
                await withTempFile(async path => {
                    console.log('Doing work with the temp file', path)
                    await writeFile(path, 'test', 'utf-8')
                    rememberPath = path
                })
            } finally {
                console.log(`Temp file cleaned up: ${
                    rememberPath != null && !await exists(rememberPath)
                }`)
            })
        })()

        // This proposal
        function tmpFile(...args) {
            return new Promise((resolve, reject) => {
                tmp.file(...args, (err, path, fd, cleanupCallback) => {
                    if (err != null) reject(err)
                    else resolve({path, fd, [Symbol.close]: cleanupCallback})
                })
            })
        }

        ;(async () => {
            let rememberPath

            try {
                with {path} = await tmpFile()
                console.log('Doing work with the temp file', path)
                await writeFile(path, 'test', 'utf-8')
                rememberPath = path
            } finally {
                console.log(`Temp file cleaned up: ${
                    rememberPath != null && !await exists(rememberPath)
                }`)
            }
        })()
        ```

    3. Multiple parallel connections

        ```js
        // Promise disposer pattern
        ;(async () => {
            await using([getConnection, getConnection], async ([conn1, conn2]) => {
                // use conn1 and conn 2 here
            })
            // Both connections closed by now
        })()

        // This proposal
        ;(async () => {
            {
                with [conn1, conn2] = await Object.useAll([
                    () => getConnection(),
                    () => getConnection(),
                ])
                // use conn1 and conn 2 here
            }
            // Both connections closed by now
        })()
        ```

    Helpers used for promise disposer variant:

    ```js
    // `using`, analogous to `Promise.using` within Bluebird
    async function using(cbs, func) {
        // Start at 1 to avoid prematurely calling `func`
        let state = {values: [], capabilities: [], remaining: 1}

        async function cont() {
            if (--state.remaining !== 0) return
            const prev = state
            state = undefined

            try {
                await func(prev.values)
                for (const p of prev.capabilities) p.resolve()
            } catch (e) {
                for (const p of prev.capabilities) p.reject(e)
            }
        }

        try {
            const p = Promise.all(Array.from(callbacks, (callback, i) => {
                if (state == null) return
                state.remaining++
                state.values[i] = undefined
                let pair
                const promise = new Promise((resolve, reject) => {
                    pair = {resolve, reject}
                })
                return cb(value => {
                    const p = pair
                    if (p == null) return promise
                    pair = undefined
                    if (state == null) {
                        p.resolve()
                    } else {
                        state.capabilities.push(p)
                        state.values[i] = value
                        cont()
                    }
                    return promise
                })
            }))

            cont()
            await p
        } catch (e) {
            const prev = state
            state = undefined
            for (const p of prev.capabilities) p.reject(e)
            throw e
        }
    }
    ```

- Simple polyfill of new additions:

    ```js
    ;(function (global) {
        "use strict"

        global.Resource = global.Resource || {}

        function newCapability() {
            let resolve, reject
            const promise = new Promise((res, rej) => { resolve = res; reject = rej })
            return {resolve, reject, promise}
        }

        global.Resource.wrap = global.Resource.wrap || init => {
            const wrap = newCapability()
            const outer = newCapability()
            const inner = newCapability()
            const createResult = value => ({value, [Symbol.close]() {
                inner.resolve()
                return outer.promise
            }})

            outer.then(() => wrap.resolve(createResult()), wrap.reject)
            try {
                outer.resolve(init(value => {
                    wrap.resolve(createResult(value))
                    return inner.promise
                }))
            } catch (e) {
                outer.reject(e)
                wrap.reject(e)
            }

            return wrap.promise
        }

        global.Resource.join = global.Resource.join || async callbacks => {
            let values = []

            values[Symbol.close] = () => {
                if (!Array.isArray(values)) return values
                return values = Promise.all(
                    values.map(async value => value[Symbol.close]())
                    .filter(value => value != null && typeof value === "function")
                )
            }

            try {
                await Promise.all(Array.from(callbacks, async callback => {
                    const value = await callback()
                    if (Array.isArray(values)) values.push(value)
                }))

                return values
            } catch (e) {
                values[Symbol.close]()
                throw e
            }
        }
    })(
        typeof window !== "undefined" ? window :
        typeof global !== "undefined" ? global :
        this
    );
    ```

- Note: https://esdiscuss.org/topic/resource-management-eg-try-with-resources#content-2

- TODO: explain how this compares to other languages
    - [Java's `try`-with-resources](https://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html)
    - [C#'s `using`](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/using-statement)
    - [Python's `with`](http://preshing.com/20110920/the-python-with-statement-by-example/)
    - [Ruby's blocks (which with `begin`/`ensure` are used more or less exactly how the disposer pattern is used)](http://jakegoulding.com/blog/2012/10/01/using-ruby-blocks-to-ensure-resources-are-cleaned-up/)
    - [C++'s](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization#C++11_example)/[Rust's](https://rustbyexample.com/std_misc/file/open.html) RAII (which this was inspired from)
