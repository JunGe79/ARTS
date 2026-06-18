# Is It Safe to Add a Virtual Method to a Base Class?

When you maintain a C++ library that other binaries link against, one
question comes up again and again:

> If I add a new `virtual` method to a base class, do I have to rebuild
> every binary that uses that class? Or can the old binaries keep running?

The short answer: **adding a new virtual method at the *end* of a class is
binary-compatible**, as long as you do not insert it in the middle and do
not change the order of the existing virtual methods. Old binaries that
were built with the old header keep working without a rebuild.

Below is a tiny experiment that proves it. Two small modules, a few lines
each, and you can run it yourself in under a minute.

## Why this works (the short version)

Every C++ object with virtual methods carries a hidden pointer to a
**vtable** — a small array of function pointers, one slot per virtual
method. The compiler turns a call like `p->canAccess(id)` into "load slot
N from the vtable and call it."

The slot number is fixed at compile time, from the header. So the rule is
simple:

- **Existing methods keep their slot numbers** as long as you do not
  reorder them or insert a new method before them.
- **A new method added at the end gets a brand new slot at the end.**

That means a binary built with the old header still finds `canAccess` and
`getCredentialById` at the exact same slots. It never knows the new slot
exists, and it never tries to use it. Nothing breaks.

The only way to get into trouble is the reverse direction: *new* code
calling the *new* method on an object whose real type was built with the
*old* header. That object's vtable has no slot for the new method, so the
call reads past the end of the array — undefined behavior. But this cannot
happen by accident, because old code does not even know the method exists.

## The experiment

We build two modules:

1. **A library** (`libbase.so`) — built with the **new** header. It does
   *not* implement the new method; it just inherits the default body from
   the base class.
2. **An app** (`app`) — built with the **old** header. It has no idea the
   new method exists. It is **not** rebuilt.

If the app runs correctly against the library, the change is
binary-compatible.

### `base_v1.h` — the OLD header

```cpp
#pragma once
#include <string>
class AdaptersInterface {
public:
    virtual ~AdaptersInterface() {}
    virtual int         canAccess(int id) = 0;
    virtual std::string getCredentialById(int id) = 0;
};
```

### `base_v2.h` — the NEW header

The only difference: a new virtual method added **at the end**, with an
inline default body so subclasses do not have to implement it.

```cpp
#pragma once
#include <string>
class AdaptersInterface {
public:
    virtual ~AdaptersInterface() {}
    virtual int         canAccess(int id) = 0;
    virtual std::string getCredentialById(int id) = 0;
    virtual long GetScaleConfigurationDetails(std::string& out) {
        out = "not implemented";
        return -1;
    }
};
```

### `lib.cpp` — the library (built with the NEW header)

Note the derived class overrides only the two old methods. It says nothing
about the new one — it just inherits the default body.

```cpp
#include "base_v2.h"

class AdaptersInterfaceDB : public AdaptersInterface {
public:
    int canAccess(int id) override { return id * 2; }
    std::string getCredentialById(int id) override {
        return "cred#" + std::to_string(id);
    }
};

extern "C" AdaptersInterface* makeDB() { return new AdaptersInterfaceDB(); }
```

### `app.cpp` — the caller (built with the OLD header)

```cpp
#include "base_v1.h"
#include <cstdio>

extern "C" AdaptersInterface* makeDB();

int main() {
    AdaptersInterface* p = makeDB();
    printf("canAccess(21)        = %d\n", p->canAccess(21));
    printf("getCredentialById(7) = %s\n", p->getCredentialById(7).c_str());
    printf("OK -- old-header binary works against new-header lib\n");
    delete p;
    return 0;
}
```

### Build and run

```sh
# library: built with the NEW header
g++ -std=c++17 -fPIC -shared lib.cpp -o libbase.so

# app: built with the OLD header, NOT rebuilt
g++ -std=c++17 app.cpp -o app -L. -lbase -Wl,-rpath,.

./app
```

Output:

```
canAccess(21)        = 42
getCredentialById(7) = cred#7
OK -- old-header binary works against new-header lib
```

The app was compiled without ever seeing `GetScaleConfigurationDetails`,
yet it links and runs correctly against a library that *was* built with the
new header. The two old methods are still at their original vtable slots,
so they resolve perfectly.

## The rules to remember

Adding a virtual method is safe **only** if you follow these rules:

1. **Add it at the very end** of the class declaration. Never insert it in
   the middle of the existing virtual methods.
2. **Do not reorder** the existing virtual methods.
3. **Do not remove** any virtual method.
4. Give it an **inline default body** if you want existing subclasses to
   compile without changes.

Break rule 1 or 2 and every existing binary needs a rebuild, because the
slot numbers shift and old code starts calling the wrong function.

One more caution: if *new* code calls the new method on an object whose
real type came from an old, un-rebuilt binary, that is undefined behavior.
In practice you avoid this by rebuilding any binary that subclasses the
base **and** has the new method called on its objects. Pure callers — code
that only receives a base pointer and calls the old methods — are always
safe and never need a rebuild.

## Takeaway

Appending a virtual method to a base class is a backward-compatible change.
You do not have to rebuild the world — only the modules that actually use
the new method on their own subclasses. Everything else keeps running on
the binaries it already has.
