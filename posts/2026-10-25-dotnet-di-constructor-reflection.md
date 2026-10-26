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

`GetConstructors()`, `GetParameters()`, and `Invoke(args)` are the whole trick. The real `Microsoft.Extensions.DependencyInjection` does exactly this.
