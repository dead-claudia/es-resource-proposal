# A proposal for resource management in JS

⚠️⚠️⚠️ This proposal is under construction. ⚠️⚠️⚠️

Things may look incomplet and incorrekt, and probably looks like a giant stage -1000 strawman. Please proceed at your own risk, as things might look completely and totally confusing to you. 😉

```
{
  use value = resource;
  use other = resource, something as other;
  
  use async value = asyncResource;
  
  use all {
    use foo = bar
    use await baz = qux
  }
}
```

- `use value = resource` - Allocate a sync resource
- `use await value = resource` - Allocate and await for an async resource
- `use all { use foo = bar; use await baz = qux; ... }` - Kind of like `Promise.all`, but handles sync errors as well
  - Note: unlike normally, the scope is actually that of `use all`, not the apparent block scope in the inside
- `value[Symbol.close]()` - Close the resource (awaited for async resources, optional)
- `use` creates `const` bindings (to make it clear you can't change what's being closed)
- `Atomics.lock(sab, index, size)` - Create a lock (this is already *technically* possible to implement, but engines can create more efficient mutex implementations here)

- Resources can be sync (e.g. locks) or async (e.g. files)
- `use` and `use async` close with the same symbol to make interop easier
- If a sync resource's `Symbol.close` method returns a thenable, an error is thrown to avoid unintentional bugs
- Usage can be sync or async (like database transactions)
- It's block scoped, so it's easier to use multiple resources at once
- It's easily optimized and trivially desugared
- It's simple to understand and easy to explain
- It plays well with the `do` expression proposal
- It's technically RAII-style, but most JS APIs already follow that to some extent.
- It avoids all the nesting issues and other functional confusion frequent with promise disposer patterns:

    ```js
    // Example 1
    // Promise disposer pattern
    const rows = await getConnection(async connection => {
       return connection.queryAsync("SELECT * FROM TABLE")
    })

    console.log(rows);

    // This proposal
    let rows = do {
      use async connection = getConnection()
      await connection.queryAsync("SELECT * FROM TABLE");
    }

    console.log(rows)

    // Example 2
    const tmp = require("tmp");
    const fs = require("fs");
    const util = require("util")

    const tmpFile = (...args) => new Promise((resolve, reject) => {
      tmp.file(...args, (err, ...ret) => err != null ? reject(err) : resolve(ret))
    })
    const exists = util.promisify(fs.exists)
    const writeFile = util.promisify(fs.writeFile)

    // Promise disposer pattern
    async function withTempFile(opts, func) {
      if (func == null) { func = opts; opts = undefined }
      const [path, fd, cleanupCallback] = await tmpFile(opts)
      try {
        return await func(path, fd)
      } finally {
        cleanupCallback()
      }
    }

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

    // This proposal
    async function withTempFile(opts) {
      const [path, fd, cleanupCallback] = await tmpFile(opts)
      return {path, fd, [Symbol.close]: cleanupCallback}
    }

    let rememberPath

    try {
      use async {path} = withTempFile()
      console.log('Doing work with the temp file', path)
      await writeFile(path, 'test', 'utf-8')
      rememberPath = path
    } finally {
      console.log(`Temp file cleaned up: ${
        rememberPath != null && !await exists(rememberPath)
      }`)
    }

    let rows = do {
      use async connection = getConnection()
      await connection.queryAsync("SELECT * FROM TABLE");
    }

    console.log(rows)
    
    // Example 3
    // Promise disposer pattern
    ;(async () => {
      await getConnection(conn1 =>
        getConnection(conn2 => {
          // use conn1 and conn 2 here
        })
      )
      // Both connections closed by now
    })()

    // This proposal
    ;(async () => {
      {
        use all {
          use async conn1 = getConnection()
          use async conn2 = getConnection()
        }
        // use conn1 and conn 2 here
      }
      // Both connections closed by now
    })()
    ```

- Note: https://esdiscuss.org/topic/resource-management-eg-try-with-resources#content-2
