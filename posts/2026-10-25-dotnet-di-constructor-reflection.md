# Who Fills the Constructor? .NET Dependency Injection and Reflection

**Question:** In an ASP.NET Core app you add a new parameter to a service's constructor — say `IPricingService` — and somehow it compiles and runs everywhere without you writing a single line to pass it in. Who actually supplies that argument, and how does the framework know what to put there?

**Answer:** The DI (dependency injection) container does. At runtime it reads the constructor's parameter *types* with reflection, looks each type up in a registration table (`interface -> implementing class`), builds an instance for each (recursively), and invokes the constructor with them. You never call `new` yourself. The only place a human fills the argument is code that bypasses the container and news the object by hand — usually a unit test.

## The setup

In ASP.NET Core you don't construct your services. You *declare what they need* as constructor parameters and *register the types once* at startup. Here is the whole loop:

```csharp
// 1. Services: an interface + its implementation.
public interface IPricingService { decimal Price(string sku); }
public class PricingService : IPricingService {
    public decimal Price(string sku) => 9.99m;
}

public interface IOrderHandler { void Checkout(string sku); }
public class OrderHandler : IOrderHandler {
    private readonly IPricingService _pricing;     // the dependency
    public OrderHandler(IPricingService pricing) { // you just DECLARE it
        _pricing = pricing;
    }
    public void Checkout(string sku) =>
        Console.WriteLine($"charge {_pricing.Price(sku)}");
}

// 2. Startup: map interface -> class. Note: no parameters are listed here.
var services = new ServiceCollection();
services.AddScoped<IPricingService, PricingService>();
services.AddScoped<IOrderHandler, OrderHandler>();
var provider = services.BuildServiceProvider();

// 3. Something asks for IOrderHandler; the container builds it, fully wired.
var handler = provider.GetRequiredService<IOrderHandler>();
handler.Checkout("ABC");   // -> "charge 9.99"
```

Notice what is *not* there: nobody wrote `new OrderHandler(new PricingService())`. You expressed a need (`OrderHandler(IPricingService pricing)`), registered the types, and `GetRequiredService` did the assembly.

The registration maps **interface to class only**. It never mentions constructor parameters. So how does the container know `OrderHandler` needs a pricing service?

## Under the hood: it's reflection

The container reads the constructor at runtime. Here is a tiny container that captures the essential move — three reflection calls:

```csharp
class MiniContainer {
    readonly Dictionary<Type, Type> _map = new();
    public void Register<TInterface, TImpl>() => _map[typeof(TInterface)] = typeof(TImpl);

    public object Resolve(Type requested) {
        var impl = _map[requested];                      // interface -> class
        var ctor = impl.GetConstructors()[0];            // REFLECTION: read the ctor
        var args = ctor.GetParameters()                  // REFLECTION: its parameter types
                       .Select(p => Resolve(p.ParameterType))  // recurse per dependency
                       .ToArray();
        return ctor.Invoke(args);                        // REFLECTION: build the object
    }
    public T Resolve<T>() => (T)Resolve(typeof(T));
}

var c = new MiniContainer();
c.Register<IPricingService, PricingService>();
c.Register<IOrderHandler, OrderHandler>();
var handler = c.Resolve<IOrderHandler>();
// reads OrderHandler's ctor -> sees IPricingService -> resolves PricingService -> injects it
```

`GetConstructors()`, `GetParameters()`, and `Invoke(args)` are the whole trick. The real `Microsoft.Extensions.DependencyInjection` does exactly this, plus the production concerns below.

## What the real container adds

- **Recursion over the whole graph.** If `PricingService` itself took an `ITaxTable`, the container would resolve that too, all the way down, before constructing anything.
- **Lifetimes.** Registration picks how long an instance lives:

  | Method | One instance per... | Typical use |
  |---|---|---|
  | `AddSingleton` | process (forever) | caches, config, stateless helpers |
  | `AddScoped` | web request | per-request handlers, DB contexts |
  | `AddTransient` | every resolve | cheap, stateless objects |

- **Caching, so it's not reflection-per-request.** It reflects a constructor the *first* time, then compiles a fast factory (an expression tree) and reuses it. Conceptually reflection; in practice compiled and cached.
- **Validation.** If a parameter's type was never registered, you get a clear runtime error ("Unable to resolve service for type ... while attempting to activate ...") instead of a mysterious null.

## The practical payoff (and the one gotcha)

Because the container fills parameters **by type**, adding a dependency is cheap and local:

```csharp
// before
public OrderHandler(IPricingService pricing) { ... }
// after -- add what you need; every call site still works, untouched
public OrderHandler(IPricingService pricing, IInventoryService inventory) { ... }
```

No caller changes anywhere — as long as `IInventoryService` is registered, the container supplies it everywhere `OrderHandler` is built.

The **one gotcha**: anything that constructs the object *by hand* bypasses the container and must be updated. In a well-structured app that's almost exclusively your **unit tests**:

```csharp
// this stops compiling the moment you add a ctor parameter
var handler = new OrderHandler(mockPricing.Object);          // missing the new arg
// fix: pass a mock for the new dependency
var handler = new OrderHandler(mockPricing.Object, mockInventory.Object);
```

That's the signal to watch for: a constructor change that "breaks the build" usually breaks *only the tests*, never production wiring — because production never calls the constructor directly.

## Takeaways

- You don't construct DI services; you declare their needs as constructor parameters and register the types once.
- The container fills each parameter by reading the constructor with **reflection** and matching parameter types to registrations, recursively.
- Registration is `interface -> class` only; the constructor signature is the source of truth for dependencies.
- Lifetimes (`Singleton`/`Scoped`/`Transient`) control how long each instance lives and how often it's rebuilt.
- Adding a constructor dependency needs **no caller changes** — except hand-written `new` calls, which in practice means your unit tests.
