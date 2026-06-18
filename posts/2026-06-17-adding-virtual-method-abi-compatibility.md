# Is It Safe to Add a Virtual Method to a Base Class?

**Question:** If I add a `virtual` method to a base class, must I rebuild
every binary that uses it?

**Answer:** No — as long as you add it at the **end** of the class and do
not reorder or remove existing methods. Old binaries keep working.

## Why

Each object with virtual methods has a hidden **vtable**: an array of
function pointers, one slot per method. A call like `p->canAccess(id)`
compiles to "call slot N." Add a method at the end and it gets a new slot
at the end — every existing method keeps its slot. Old code still finds
`canAccess` where it expects, and never touches the new slot.

## Proof in two modules

**Library** built with the **new** header (does not even implement the new
method — inherits the default body):

```cpp
// base_v2.h
class AdaptersInterface {
public:
    virtual ~AdaptersInterface() {}
    virtual int         canAccess(int id) = 0;
    virtual std::string getCredentialById(int id) = 0;
    virtual long GetScaleConfigurationDetails(std::string& out) {  // NEW, at end
        out = "not implemented"; return -1;
    }
};

// lib.cpp
class AdaptersInterfaceDB : public AdaptersInterface {
    int canAccess(int id) override { return id * 2; }
    std::string getCredentialById(int id) override { return "cred#" + std::to_string(id); }
};
extern "C" AdaptersInterface* makeDB() { return new AdaptersInterfaceDB(); }
```

**App** built with the **old** header (no new method, not rebuilt):

```cpp
// base_v1.h  -- same class WITHOUT GetScaleConfigurationDetails
// app.cpp
extern "C" AdaptersInterface* makeDB();
int main() {
    AdaptersInterface* p = makeDB();
    printf("%d\n",  p->canAccess(21));          // 42
    printf("%s\n",  p->getCredentialById(7).c_str());  // cred#7
    delete p;
}
```

Build and run:

```sh
g++ -std=c++17 -fPIC -shared lib.cpp -o libbase.so
g++ -std=c++17 app.cpp -o app -L. -lbase -Wl,-rpath,.
./app          # 42 / cred#7  -- works, no rebuild
```

The app was compiled without ever seeing the new method, yet runs fine
against the new library.

## Rules

1. Add the new method at the **very end** of the class.
2. Never **reorder** or **remove** existing virtual methods.
3. Give it an **inline default body** so existing subclasses still compile.

Break rule 1 or 2 and the slot numbers shift — then every binary must be
rebuilt.

One caveat: do not call the *new* method on an object whose real type came
from an old, un-rebuilt binary (its vtable has no such slot — undefined
behavior). Pure callers that only use the old methods are always safe.
