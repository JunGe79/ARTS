# Tampering a C++ string argument at runtime with Frida

**Question:** You fixed a server so it rejects bad input (say, an upload file name with `../` in it). How do you prove the fix end to end, when the bad input has to arrive from a *client* that never sends anything bad on purpose?

**Answer:** Don't recompile the client. Attach [Frida](https://frida.re) to the running client, hook the function that sends the value, and rewrite the argument in memory right before it goes out. The real client then sends your malicious value on the wire, and you watch the real server react.

This post shows the technique on a tiny program you can run yourself in five minutes.

## The setup

I had a server-side fix that was supposed to *fail closed*: if a client sends an upload name containing `/`, `\`, or `..`, reject it instead of trying to clean it up. The unit tests passed. But I wanted the real thing: a real client process sending a real `../../../x` over the real protocol, hitting the real compiled server.

The obvious way — patch the client to send a bad name — means rebuilding and redeploying the client. Slow, and it changes the binary you're testing. Frida is better: it leaves every binary untouched and changes one function argument in live memory.

The client's send function looked like this:

```cpp
void sendFile(const std::string& handle,
              const std::string& fullPath,
              const std::string& fileName);  // <-- I want to tamper this one
```

`fileName` is what the server stores. If I can rewrite that `std::string` in memory on the way in, the client does the rest for me.

## What Frida is, in one paragraph

Frida injects a small JavaScript engine into a running process. Your JS can find functions, hook them (`Interceptor.attach`), and read or write any memory in the process. You attach to a process that is already running, or spawn a new one under instrumentation. No source, no recompile, no debugger protocol to wrangle.

## The core move: hook a function and edit an argument

```js
const addr = Module.getGlobalExportByName("sendFile");   // resolve the function
Interceptor.attach(addr, {
  onEnter(args) {
    // args[0], args[1], ... are the raw argument registers/pointers.
    // For a C++ *member* function, args[0] is `this` and the real args shift by one.
    const fileName = args[2];     // the 3rd argument: const std::string&
    // ... rewrite the std::string that fileName points at ...
  }
});
```

`onEnter` runs *before* the function body, so whatever you write is what the function sees. That's the whole idea. The hard part is the next section: a `std::string` is not a `char*`, and you have to edit it correctly.

## Reading and writing a `std::string` in memory

A `std::string` is a small struct, and its layout depends on the standard library. You must know the layout to edit one.

**libgcc / libstdc++ (Linux, GCC/Clang), 64-bit:**

| offset | field |
|---|---|
| 0 | `char*` pointer to the character data |
| 8 | size (length) |
| 16 | 16-byte inline buffer **or** heap capacity |

The pointer at offset 0 *always* points at the data. For a short string (≤ 15 chars) the data lives inline, so the pointer points back into the object itself at offset 16; capacity is a fixed 15. This is the "small string optimization" (SSO).

**MSVC (Windows), 64-bit** is different: a 16-byte union at offset 0 (inline buffer *or* a heap pointer), size at offset 16, capacity at offset 24. `c_str()` returns the inline buffer only when capacity < 16.

Two rules that save you a lot of pain:

1. **Edit in place; stay within capacity.** Overwrite the existing buffer and fix the size field. A short payload fits the 15-byte inline buffer; a longer one needs the string to already be heap-allocated with enough capacity.
2. **Do not swap the data pointer to your own buffer.** It's tempting to `Memory.alloc` a bigger buffer and point the string at it. Don't. The string still *thinks* it owns its storage — when it goes out of scope, its destructor will `free()` your pointer. On some libraries the swap silently doesn't stick; on others it corrupts the heap. In-place-within-capacity is the safe path.

Here's the libstdc++ version of the edit:

```js
const obj  = args[2];                        // const std::string&
const dptr = obj.readPointer();              // data pointer (offset 0)
const size = obj.add(8).readU64().valueOf(); // length (offset 8)
const isSSO = dptr.equals(obj.add(16));      // inline buffer?
const cap   = isSSO ? 15 : obj.add(16).readU64().valueOf();

if (PAYLOAD.length <= cap) {
  dptr.writeUtf8String(PAYLOAD);             // writes bytes + a NUL terminator
  obj.add(8).writeU64(PAYLOAD.length);       // fix the length so size() is right
}
```

That's it. When the function reads `fileName.c_str()` and `fileName.length()`, it gets your payload.

## Practice experiment (about five minutes)

You don't need a server or a special target. Here is a self-contained program that calls a function with a `std::string` in a loop. You'll hook it and watch the printed value change — proof that the argument was rewritten in memory.

**1. The target.** Save as `target.cpp`:

```cpp
#include <string>
#include <iostream>
#include <thread>
#include <chrono>

// Stand-in for "the client function that sends a file name".
extern "C" void uploadFile(const std::string& name) {
    std::cout << "uploading [" << name << "]" << std::endl;
}

int main() {
    for (int i = 0; ; ++i) {
        uploadFile("report_" + std::to_string(i) + ".txt");
        std::this_thread::sleep_for(std::chrono::seconds(1));
    }
}
```

`extern "C"` keeps the name un-mangled so Frida can find it by `uploadFile`. `-rdynamic` puts it in the dynamic symbol table.

```sh
g++ -O0 -g -rdynamic -pthread -o target target.cpp
./target        # prints: uploading [report_0.txt], [report_1.txt], ...
```

**2. The hook.** Save as `hook.js`:

```js
// libstdc++ std::string (64-bit): [0]=char* data, [8]=size, [16]=SSO buffer / heap capacity.
const PAYLOAD = "../../../ZZPWN";                    // 14 chars; fits the 15-byte inline buffer
const addr = Module.getGlobalExportByName("uploadFile");
console.log("[hook] uploadFile @ " + addr);
Interceptor.attach(addr, {
  onEnter(args) {
    const obj   = args[0];                           // const std::string& name
    const dptr  = obj.readPointer();
    const size  = obj.add(8).readU64().valueOf();
    const cur   = dptr.readUtf8String(size);
    const isSSO = dptr.equals(obj.add(16));
    const cap   = isSSO ? 15 : obj.add(16).readU64().valueOf();
    if (PAYLOAD.length <= cap) {
      dptr.writeUtf8String(PAYLOAD);
      obj.add(8).writeU64(PAYLOAD.length);
      console.log("[hook] [" + cur + "] cap=" + cap + " -> [" + PAYLOAD + "]");
    } else {
      console.log("[hook] SKIP: payload " + PAYLOAD.length + " > capacity " + cap);
    }
  }
});
```

**3. Attach.** In another terminal (`pip install frida-tools`):

```sh
frida -n target -l hook.js
```

Watch the first terminal. The moment Frida attaches, the output flips:

```
uploading [report_3.txt]
uploading [report_4.txt]
uploading [../../../ZZPWN]      <-- hook attached here
uploading [../../../ZZPWN]
```

The program never knew. You changed one argument in flight.

**4. Hit the capacity wall on purpose.** Change the payload to something long:

```js
const PAYLOAD = "../../../../../../etc/shadow";   // 28 chars, bigger than the 15-byte inline buffer
```

Re-run. Now you get `SKIP: payload 28 > capacity 15`, and the target keeps printing normal names. This is the real constraint from the top of the post: a short string only has room for ~15 bytes, so your tamper payload has to fit. Pick a shorter payload, or find an injection point where the string is already large (a long path, a big buffer) and therefore heap-allocated with spare capacity.

## Gotchas worth knowing before you spend an hour on them

- **`import frida` crashes on startup?** If there's a stray `.py` file in your working directory shadowing a standard-library module, the native `_frida` import can fail with a cryptic `SystemError`. Run Python in isolated mode: `python -I`. It drops the current directory from the import path.
- **Member vs free function.** For a C++ *method*, `args[0]` is the hidden `this`, so your first real argument is `args[1]`. For a plain function (like `uploadFile`), the first argument is `args[0]`. Off-by-one here reads garbage.
- **Finding a mangled name.** A normal C++ function is name-mangled. Dump the exports (`nm -D`, or `dumpbin /EXPORTS` on Windows) and pass the exact mangled symbol to the resolver. Or make the function `extern "C"` in a test target like we did.
- **Log to a file, not the console,** if you hook something that runs for a long time over a flaky connection. Console output over a remote shell can buffer and arrive late or out of order.
- **Kill your old hook before starting a new one.** Two Frida sessions attached to the same function will both fire and fight over the same argument. If you background a hook process, make sure it's actually dead before the next run.

## Takeaways

- Frida lets you test a server's input handling by tampering what a *real* client sends, with no rebuild of the client. That's the closest thing to the real attack you can get.
- To edit a `std::string` you must know its memory layout. libstdc++ puts the data pointer first; MSVC uses a union and decides inline-vs-heap by capacity. Learn the three fields: data, size, capacity.
- Rewrite in place and stay within capacity. Never swap the data pointer to your own buffer — the destructor will try to free it.
- The practice target above is a safe way to get all of this working before you point Frida at anything that matters.
