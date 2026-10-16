

  

## Detailed study notes with implementation, engineering practice, and interview preparation

  

**Source:** Transcript supplied for the video *"Design Patterns were

hard until I learned these concepts \| Design Patterns 101: Creational

Patterns"*.

  

**Scope note:** The video introduces design patterns, then focuses on

four creational patterns: Singleton, Factory Method (presented as a

dedicated factory/helper), Abstract Factory, and Builder. It briefly

mentions Prototype but does not teach it in depth. The implementation

examples below are practical companion examples written in Python; they

expand on the transcript and are labelled as implementation notes rather

than claims that the video used Python.

  

------------------------------------------------------------------------

  

# 1. What is a design pattern?

  

A design pattern is a reusable design idea for a problem that repeatedly

appears while building software. It is **not a copy-paste code

snippet**. You adapt the idea to the language, architecture,

constraints, and team conventions of your project.

  

Patterns are useful because they can: - reduce repeated ad-hoc

solutions; - make code easier to extend and maintain; - give engineers

shared vocabulary ("use a Factory here" communicates a design direction

quickly); - reduce the chance that adding a new type forces changes in

many unrelated places; - make code easier for another developer to

understand.

  

A pattern does not automatically make code good. Applying a pattern

where the problem does not need it can add unnecessary classes,

indirection, and complexity.

  

## 1.1 The three broad families

  

  -----------------------------------------------------------------------

  Family                  Main question           Examples

  ----------------------- ----------------------- -----------------------

  Creational              How should objects be   Singleton, Factory

                          created?                Method, Abstract

                                                  Factory, Builder,

                                                  Prototype

  

  Structural              How should              Adapter, Decorator,

                          objects/classes be      Facade

                          composed?              

  

  Behavioral              How should objects      Observer, Strategy,

                          communicate or          Visitor

                          distribute              

                          responsibility?        

  -----------------------------------------------------------------------

  

A quick diagnostic: - If the central issue is *which object to create,

how many to create, or how to assemble it*, investigate a creational

pattern. - If the issue is *how to connect or wrap components*,

investigate a structural pattern. - If the issue is *how to coordinate

behavior or vary an algorithm*, investigate a behavioral pattern.

  

This is a heuristic, not a formal rule: real designs can involve

multiple pattern families.

  

------------------------------------------------------------------------

  

# 2. Singleton

  

## 2.1 The problem

  

Suppose multiple parts of an application need to write diagnostic events

to one logging destination. If each component creates its own logger and

opens the same file independently, the program may have: - multiple file

handles; - inconsistent formatting or configuration; - interleaved

writes; - duplicated resources; - difficult-to-reason-about lifecycle

and shutdown behavior.

  

The Singleton pattern aims to provide **one shared instance through one

well-known access point**.

  

Typical candidates may include a process-wide configuration object, a

carefully designed metrics registry, or a shared in-process resource

coordinator. These are candidates, not automatic reasons to use

Singleton.

  

## 2.2 The structure

  

A classic Singleton has: 1. **Singleton class:** the class whose

instance is controlled. 2. **Private constructor (in languages that

support it):** prevents callers from constructing arbitrary instances.

3. **Static/shared access method:** returns the instance without

requiring a caller to already have one. 4. **Stored instance:** holds

the object once initialized.

  

Conceptual UML:

  

``` text

+------------------------------+

| Logger                       |

+------------------------------+

| - instance: Logger (static)  |

| - constructor()              |

+------------------------------+

| + get_instance(): Logger     |

| + log(message): void         |

+------------------------------+

```

  

UML visibility symbols: - `+` public - `-` private - `#` protected

  

## 2.3 Python implementation

  

Python does not enforce private constructors in the same way as C++ or

Java. A simple module-level instance is often more idiomatic than a

hand-built Singleton. This example shows the classic idea for learning

purposes:

  

``` python

from threading import Lock

  

class Logger:

    _instance = None

    _lock = Lock()

  

    def __new__(cls):

        # Protect instance creation when multiple threads call at once.

        with cls._lock:

            if cls._instance is None:

                cls._instance = super().__new__(cls)

                cls._instance._initialized = False

        return cls._instance

  

    def __init__(self):

        # __init__ may run on each construction attempt, so guard setup.

        if self._initialized:

            return

  

        self._initialized = True

        self._messages = []

  

    def log(self, message: str) -> None:

        self._messages.append(message)

  

    def messages(self) -> list[str]:

        return list(self._messages)

  
  

a = Logger()

b = Logger()

  

assert a is b

a.log("processing order")

b.log("processing payment")

assert a.messages() == ["processing order", "processing payment"]

```

  

This demonstrates one instance within this Python process. It is an

educational in-memory logger, not a production logging system. Real

logging libraries generally provide more complete handling of

concurrency, rotation, formatting, buffering, and shutdown.

  

### Why the lock matters

  

Without synchronization, two threads can both observe

`_instance is None` and each attempt to initialize an instance. The

check-and-create operation must be protected if it can run concurrently.

  

A lock around initialization is one approach. Language-specific

mechanisms may be safer or simpler: - C++: function-local static

initialization is thread-safe in modern C++. - Java: an enum Singleton

or initialization-on-demand holder can be robust options. - Python: a

module-level object or dependency injection may be simpler than a custom

Singleton.

  

Do not assume that every Singleton implementation is thread-safe merely

because it is called a Singleton.

  

## 2.4 Usage

  

``` python

logger = Logger()

logger.log("payment request received")

```

  

Any later `Logger()` call in this example returns the same object.

  

## 2.5 Critical real-world caveats

  

**One instance in what scope?** Usually the guarantee is only within one

process and one class-loading/runtime context. It does not automatically

mean one instance across: - multiple server processes; - multiple

machines; - containers or replicas; - separate Python interpreters; -

independently loaded class copies.

  

If multiple service replicas must coordinate, a distributed system

mechanism (such as a database constraint, distributed lock, or

single-writer service) may be needed. An in-process Singleton cannot

provide a global distributed guarantee.

  

**Singleton is not the same as thread safety.** The object can be unique

while its methods still have race conditions. Protect shared mutable

state separately.

  

**Singleton is not automatically the best logger design.** Dependency

injection of a logger is often easier to test. Standard logging

frameworks already manage shared handlers and concurrency.

  

**Global state makes tests harder.** Tests may leak state into each

other, become order-dependent, or need special reset hooks. Prefer

explicit dependencies when practical.

  

**Lifecycle matters.** Ask who flushes buffered output, closes

resources, handles errors, and shuts down the instance.

  

## 2.6 When to use / avoid

  

Use when: - exactly one instance per relevant process/scope is a genuine

invariant; - the shared object has a clear lifecycle; - global access is

an acceptable trade-off.

  

Avoid when: - ordinary dependency injection would work; - callers need

different configurations or isolated instances; - tests need independent

state; - a distributed "single instance" is being assumed without

distributed coordination.

  

## 2.7 Tests to write

  

-   Two access attempts return the same object (`is`, not merely equal

    values).

-   Initialization occurs once.

-   Concurrent first access does not create multiple instances.

-   Methods remain correct under concurrent use if concurrency is

    supported.

-   Resource failures and shutdown behavior are handled.

  

------------------------------------------------------------------------

  

# 3. Factory Method / dedicated Factory

  

## 3.1 The problem

  

Imagine a trading application that can create different order types: -

Market order: execute at the available market price. - Limit order:

execute only at a specified price or better.

  

A naive design repeats this decision everywhere:

  

``` python

if order_type == "market":

    order = MarketOrder()

elif order_type == "limit":

    order = LimitOrder()

else:

    raise ValueError("Unsupported order type")

```

  

At one call site, this may be fine. If the same decision is repeated

across dozens of modules, it causes: - duplicated conditionals; -

inconsistent validation; - spelling/string mistakes; - many locations to

update when a type is added or removed; - business code becoming coupled

to concrete implementation classes.

  

The video's practical message is to centralize object creation in a

dedicated helper/factory.

  

> Terminology precision: the video uses "Factory Method" for this

> central factory-helper example. In the classic GoF definition,

> **Factory Method** specifically relies on a creator interface/class

> whose subclasses override a creation method. A central class with a

> `create()` method and a conditional is often called a **Simple

> Factory**. The core motivation---centralizing creation---is similar,

> but the structures are not identical.

  

## 3.2 Roles and vocabulary

  

-   **Product:** the common abstraction, such as `Order`.

-   **Concrete products:** `MarketOrder`, `LimitOrder`.

-   **Creator/factory:** the component responsible for

    selecting/creating the product.

-   **Client:** the code requesting an order without needing to

    construct its concrete class directly.

  

## 3.3 Practical Python implementation

  

``` python

from abc import ABC, abstractmethod

  

class Order(ABC):

    @abstractmethod

    def execute(self) -> str:

        pass

  
  

class MarketOrder(Order):

    def execute(self) -> str:

        return "Market order submitted"

  
  

class LimitOrder(Order):

    def __init__(self, limit_price: float):

        if limit_price <= 0:

            raise ValueError("limit_price must be positive")

        self.limit_price = limit_price

  

    def execute(self) -> str:

        return f"Limit order submitted at {self.limit_price}"

  
  

class OrderFactory:

    @staticmethod

    def create(order_type: str, **kwargs) -> Order:

        if order_type == "market":

            return MarketOrder()

        if order_type == "limit":

            return LimitOrder(limit_price=kwargs["limit_price"])

        raise ValueError(f"Unsupported order type: {order_type}")

  
  

order = OrderFactory.create("market")

print(order.execute())

  

limit = OrderFactory.create("limit", limit_price=125.50)

print(limit.execute())

```

  

The client depends on the `Order` abstraction and the factory entry

point, rather than duplicating concrete construction logic.

  

### Production improvements

  

For a small fixed set of types, a conditional is reasonable. For a

plugin-based system, use a registry or classic polymorphic Factory

Method so adding a new type does not require editing a giant

conditional. Validate input at boundaries and do not let untrusted user

input instantiate arbitrary classes.

  

Example registry approach:

  

``` python

class RegisteredOrderFactory:

    _creators = {

        "market": lambda **kw: MarketOrder(),

        "limit": lambda **kw: LimitOrder(limit_price=kw["limit_price"]),

    }

  

    @classmethod

    def register(cls, name, creator):

        if name in cls._creators:

            raise ValueError(f"Order type already registered: {name}")

        cls._creators[name] = creator

  

    @classmethod

    def create(cls, name, **kwargs):

        try:

            creator = cls._creators[name]

        except KeyError:

            raise ValueError(f"Unsupported order type: {name}") from None

        return creator(**kwargs)

```

  

A registry is one implementation option, not automatically superior. It

introduces mutable registration state and should be managed

deliberately.

  

## 3.4 Classic Factory Method vs Simple Factory

  

Classic Factory Method:

  

``` text

Creator

  └── create_product()  [creation hook]

       ↑ overridden by

ConcreteCreatorA → ProductA

ConcreteCreatorB → ProductB

```

  

The creator subclasses decide which concrete product to instantiate.

This is different from a single `OrderFactory.create(type)` method that

switches on a string.

  

Use the classic version when subclasses genuinely need to vary the

construction decision. Use a simple factory when a centralized creation

function is sufficient.

  

## 3.5 Why it helps

  

Without a factory, 50 call sites might each need modification when an

order type changes. With centralized creation, callers use one stable

API and the construction decision is managed in one place.

  

However, a factory does not magically eliminate all changes: adding a

new product can still require updating a registry, configuration, tests,

and domain rules. The benefit is reducing duplicated creation logic and

coupling.

  

## 3.6 Testing checklist

  

-   Each supported type returns the correct implementation.

-   Unsupported types fail clearly.

-   Required parameters are validated.

-   Each returned product satisfies the shared interface.

-   Client code works through the abstraction and does not depend on

    concrete classes unnecessarily.

  

------------------------------------------------------------------------

  

# 4. Abstract Factory

  

## 4.1 The problem: products that must match

  

The video's example is a trading system supporting different exchanges.

Each exchange has multiple connector types: - an order connector; - a

market-data connector; - potentially more connector types later.

  

For example: - NYSE order connector must pair with NYSE market-data

connector. - NASDAQ order connector must pair with NASDAQ market-data

connector.

  

Creating these products independently makes mismatched combinations

possible. A connector pair might compile successfully while speaking

incompatible protocol dialects, leading to runtime failures or incorrect

data interpretation.

  

**Abstract Factory provides an interface for creating a family of

related products while keeping the family consistent.**

  

## 4.2 Roles

  

-   **Abstract factory:** declares creation methods for each product

    type.

-   **Concrete factory:** creates a matching family, e.g. NYSE or

    NASDAQ.

-   **Abstract products:** interfaces such as `OrderConnector` and

    `MarketDataConnector`.

-   **Concrete products:** exchange-specific implementations.

-   **Client:** asks a factory for products without directly choosing

    each concrete class.

  

Conceptual structure:

  

``` text

ExchangeConnectorFactory

  + create_order_connector()

  + create_market_data_connector()

           ↑

     ┌─────┴─────┐

 NYSEFactory  NasdaqFactory

     │              │

 NYSE order       Nasdaq order

 NYSE market data Nasdaq market data

```

  

The essential property is that both products are created by the same

selected family factory.

  

## 4.3 Python implementation

  

``` python

from abc import ABC, abstractmethod

  

class OrderConnector(ABC):

    @abstractmethod

    def send_order(self, order_id: str) -> str:

        pass

  
  

class MarketDataConnector(ABC):

    @abstractmethod

    def receive_data(self) -> str:

        pass

  
  

class NyseOrderConnector(OrderConnector):

    def send_order(self, order_id: str) -> str:

        return f"Sending order {order_id} through NYSE"

  
  

class NyseMarketDataConnector(MarketDataConnector):

    def receive_data(self) -> str:

        return "Receiving NYSE market data"

  
  

class NasdaqOrderConnector(OrderConnector):

    def send_order(self, order_id: str) -> str:

        return f"Sending order {order_id} through NASDAQ"

  
  

class NasdaqMarketDataConnector(MarketDataConnector):

    def receive_data(self) -> str:

        return "Receiving NASDAQ market data"

  
  

class ExchangeConnectorFactory(ABC):

    @abstractmethod

    def create_order_connector(self) -> OrderConnector:

        pass

  

    @abstractmethod

    def create_market_data_connector(self) -> MarketDataConnector:

        pass

  
  

class NyseFactory(ExchangeConnectorFactory):

    def create_order_connector(self) -> OrderConnector:

        return NyseOrderConnector()

  

    def create_market_data_connector(self) -> MarketDataConnector:

        return NyseMarketDataConnector()

  
  

class NasdaqFactory(ExchangeConnectorFactory):

    def create_order_connector(self) -> OrderConnector:

        return NasdaqOrderConnector()

  

    def create_market_data_connector(self) -> MarketDataConnector:

        return NasdaqMarketDataConnector()

  
  

def connect_to_exchange(factory: ExchangeConnectorFactory):

    order_connector = factory.create_order_connector()

    market_data_connector = factory.create_market_data_connector()

  

    # Client talks only to abstractions.

    print(order_connector.send_order("ORD-101"))

    print(market_data_connector.receive_data())

  
  

connect_to_exchange(NyseFactory())

connect_to_exchange(NasdaqFactory())

```

  

The client does not need `if exchange == ...` checks for every connector

type. The chosen factory supplies a consistent family.

  

## 4.4 Important limitation: consistency is an architectural guarantee, not magic

  

Abstract Factory makes mismatching products harder by construction, but

it cannot guarantee that: - the exchange's credentials are valid; - the

network is reachable; - the remote server is healthy; - the protocol

implementation has no bugs; - the products are configured with

compatible runtime versions if configuration bypasses the factory.

  

For a real trading connector, also validate configuration, protocol

versions, authentication, timeouts, retries, rate limits, observability,

and error handling.

  

If the product constructors need credentials or connection pools, inject

those dependencies into the concrete factory rather than using

hard-coded secrets.

  

## 4.5 When Abstract Factory is useful

  

-   multiple database vendors with related connection, query, and

    transaction components;

-   cloud-provider SDK families;

-   UI themes where buttons, menus, and checkboxes must be visually

    consistent;

-   exchange integrations where multiple connectors must use the same

    protocol family;

-   storage backends with a coordinated group of reader, writer, and

    metadata clients.

  

## 4.6 When it is overkill

  

If the application only needs one object type and there are no related

product families to keep consistent, a simple factory may be enough.

Abstract Factory creates more interfaces and classes, so use it when

family-level variation is a real requirement.

  

------------------------------------------------------------------------

  

# 5. Factory vs Abstract Factory --- the crucial distinction

  

  ------------------------------------------------------------------------------

  Question             Simple Factory /       Abstract Factory

                       Factory Method family  

  -------------------- ---------------------- ----------------------------------

  Main goal            Choose/create one      Create a compatible family of

                       product                products

  

  Example              Market order vs limit  NYSE order connector plus NYSE

                       order                  market-data connector

  

  Typical creation API `create_order(type)`   `create_order_connector()` and

                                              `create_market_data_connector()`

  

  Main variation       One product            Multiple product abstractions vary

                       abstraction has        together

                       different              

                       implementations        

  

  Main risk addressed  Duplicated creation    Mismatched combinations of related

                       decisions              components

  ------------------------------------------------------------------------------

  

**Fast interview heuristic:** count the product families, not just the

number of concrete classes. A single factory can have multiple branches

and still be a simple factory. Abstract Factory is characterized by

creating a family of related product types.

  

------------------------------------------------------------------------

  

# 6. Builder

  

## 6.1 The problem: constructors with many arguments

  

An HTTP request might include: - URL; - method; - body; - headers; -

timeout; - retry policy; - authentication; - tracing information; -

proxy configuration.

  

A long positional constructor is hard to read:

  

``` python

request = HttpRequest(

    "https://api.example.com/orders",

    "POST",

    '{"symbol":"ABC"}',

    5000,

    True,

    False,

)

```

  

The reader may not know which boolean means what. Two arguments with the

same type can be accidentally swapped and still pass basic type

checking. Optional parameters make the constructor harder to understand

and maintain.

  

**Builder constructs a complex object step by step using named, readable

methods, then returns the completed object from `build()`.**

  

## 6.2 Python implementation

  

This example uses an immutable request object and a mutable builder. It

validates the request at build time.

  

``` python

from dataclasses import dataclass

from typing import Mapping

  

@dataclass(frozen=True)

class HttpRequest:

    url: str

    method: str

    headers: Mapping[str, str]

    body: str | None

    timeout_ms: int

  
  

class HttpRequestBuilder:

    def __init__(self):

        self._url = None

        self._method = "GET"

        self._headers = {}

        self._body = None

        self._timeout_ms = 5000

  

    def url(self, url: str):

        self._url = url

        return self

  

    def method(self, method: str):

        self._method = method.upper()

        return self

  

    def header(self, name: str, value: str):

        self._headers[name] = value

        return self

  

    def body(self, body: str):

        self._body = body

        return self

  

    def timeout_ms(self, timeout_ms: int):

        if timeout_ms <= 0:

            raise ValueError("timeout_ms must be positive")

        self._timeout_ms = timeout_ms

        return self

  

    def build(self) -> HttpRequest:

        if not self._url:

            raise ValueError("URL is required")

  

        if self._method not in {"GET", "POST", "PUT", "PATCH", "DELETE"}:

            raise ValueError("Unsupported HTTP method")

  

        if self._method == "GET" and self._body is not None:

            raise ValueError("This example does not allow a GET body")

  

        return HttpRequest(

            url=self._url,

            method=self._method,

            headers=dict(self._headers),

            body=self._body,

            timeout_ms=self._timeout_ms,

        )

  
  

request = (

    HttpRequestBuilder()

    .url("https://api.example.com/orders")

    .method("POST")

    .header("Content-Type", "application/json")

    .body('{"symbol":"ABC"}')

    .timeout_ms(3000)

    .build()

)

  

print(request)

```

  

This demonstrates the transcript's chainable API: each configuration

method returns the builder itself, so the next method can be called. The

final `build()` method returns the actual `HttpRequest`.

  

This example only constructs a request. It does not send a network

request.

  

## 6.3 How the builder works

  

1.  Create a builder with defaults.

2.  Call named methods to set the fields required for this object.

3.  Validate cross-field constraints in `build()`.

4.  Construct and return the final object.

5.  Use the returned object independently of the builder.

  

The `HttpRequest` above is frozen, so its fields cannot be reassigned

normally after creation. The builder remains mutable during assembly.

  

## 6.4 Production implementation notes

  

**Required fields:** validate them in `build()`, or require them in the

builder constructor if that makes invalid states harder to represent.

  

**Defaults:** make defaults explicit and documented. Avoid hidden

defaults that can change request semantics unexpectedly.

  

**Cross-field validation:** for example, an authenticated request may

require credentials; a body may require a content type; a retry policy

may not be suitable for a non-idempotent operation.

  

**Defensive copies:** the example copies headers with

`dict(self._headers)`. Without a copy, later changes to the builder

could affect the already-built request if the same mutable dictionary

were shared.

  

**Builder reuse:** decide whether a builder can build several objects or

should be discarded after one build. Reuse can accidentally carry

headers or body data from one request into the next. A fresh builder per

request is a simple safe default.

  

**Secret handling:** never log authorization tokens or sensitive headers

as part of debugging output.

  

**Immutability:** an immutable built object is easier to share safely,

cache, and reason about. `frozen=True` is shallow; nested mutable values

still need care.

  

**Do not build a builder for every class.** For an object with two

simple fields, a normal constructor or keyword arguments are often

clearer.

  

## 6.5 Builder versus setters

  

A public mutable object with setters allows callers to create partially

configured objects and forget required steps. A Builder can centralize

validation and make the transition from incomplete configuration to a

valid finished object explicit.

  

However, setters can be perfectly reasonable for genuinely mutable

domain objects. Builder is especially useful when construction has many

optional fields, meaningful defaults, or cross-field validation.

  

------------------------------------------------------------------------

  

# 7. Prototype --- briefly mentioned in the video

  

The transcript mentions Prototype as the fifth creational pattern but

does not explore it in detail.

  

Prototype creates a new object by copying an existing object rather than

building one from scratch.

  

Example:

  

``` python

from copy import deepcopy

  

template = {

    "theme": "dark",

    "columns": ["name", "status"],

    "page_size": 25,

}

  

new_config = deepcopy(template)

new_config["columns"].append("owner")

  

assert template["columns"] == ["name", "status"]

```

  

The deep copy is important here because the object contains a nested

list. A shallow copy would create a new outer dictionary but could share

the inner list.

  

Real-world uses include copying configuration templates, document

templates, or preconfigured objects. Copying objects with file handles,

sockets, database connections, locks, or other external resources

requires special care; a generic copy may not produce a valid

independent resource.

  

------------------------------------------------------------------------

  

# 8. Practical comparison: which pattern should I choose?

  

Ask these questions in order:

  

1.  **Must there be one shared instance?** Consider Singleton, but first

    evaluate dependency injection and lifecycle.

2.  **Do I need to choose one implementation of one product?** Consider

    a simple factory or Factory Method.

3.  **Must several different product types be created as a matching

    family?** Consider Abstract Factory.

4.  **Does the object have many optional fields or complex construction

    rules?** Consider Builder.

5.  **Do I already have a suitable object and need a copy?** Consider

    Prototype.

  

  -----------------------------------------------------------------------

  Scenario                Likely fit              Reason

  ----------------------- ----------------------- -----------------------

  Central process-wide    Singleton or injected   One shared

  configuration           configuration           configuration source

                                                  may be required

  

  Market vs limit order   Factory                 One product

                                                  abstraction, several

                                                  implementations

  

  Exchange-specific order Abstract Factory        Products must belong to

  and market-data clients                         one compatible family

  

  HTTP request with       Builder                 Named steps and

  optional headers, body,                         validation improve

  timeout, retries                                construction

  

  Copying a dashboard     Prototype/copying       Start from an existing

  configuration template                          object

  -----------------------------------------------------------------------

  

The word "likely" matters: these are starting points, not automatic

rules.

  

------------------------------------------------------------------------

  

# 9. Common mistakes and how to avoid them

  

## Singleton

  

-   Assuming one object means thread-safe methods.

-   Assuming one process-wide object means one object across the whole

    distributed system.

-   Using it for every shared service and creating hidden global

    dependencies.

-   Ignoring initialization failure, shutdown, or test isolation.

  

## Factory

  

-   Keeping the same large conditional in many different files.

-   Returning unrelated types without a common interface.

-   Silently defaulting to an unintended product for unknown input.

-   Treating every conditional as a reason to introduce a factory.

-   Confusing a Simple Factory with the classic GoF Factory Method

    structure.

  

## Abstract Factory

  

-   Using it when only one product type varies.

-   Accidentally bypassing the factory and constructing products

    independently.

-   Assuming a family factory validates network, credential, or protocol

    health.

-   Building a large hierarchy that adds more complexity than it

    removes.

  

## Builder

  

-   Forgetting to validate required fields in `build()`.

-   Sharing mutable collections between the builder and final object.

-   Allowing builder state to leak between requests.

-   Adding builder classes for trivial objects.

-   Treating method names as a substitute for validation.

  

------------------------------------------------------------------------

  

# 10. Interview-ready explanations

  

### Singleton

  

"Singleton controls instance creation so that a defined scope has one

shared instance and a common access point. A logger is a common example.

I would also discuss thread safety, lifecycle, testability, and the fact

that a process-local Singleton does not coordinate across distributed

replicas."

  

### Factory

  

"A factory centralizes the decision about which concrete implementation

to construct. In a trading application, the client can request a market

or limit order through a common `Order` abstraction instead of repeating

concrete construction logic throughout the codebase."

  

### Abstract Factory

  

"Abstract Factory creates families of related products. For example, an

exchange-specific factory can create both an order connector and a

market-data connector for NYSE or both for NASDAQ, reducing the chance

of mixing incompatible families."

  

### Builder

  

"Builder assembles an object step by step through named methods and

returns the completed object from `build()`. It is useful when there are

many optional fields or complex validation, because it improves

readability and can prevent invalid configurations."

  

### Difference between Factory and Abstract Factory

  

"Factory chooses one product implementation. Abstract Factory creates

multiple related product types as a compatible family."

  

### Why not use patterns everywhere?

  

"Patterns are tools for recurring design problems. If a normal

constructor or a small function is already clear and maintainable,

adding pattern classes may be unnecessary abstraction."

  

------------------------------------------------------------------------

  

# 11. Practice exercises

  

Implement these without copying the examples first.

  

## Exercise A --- Singleton

  

Create a configuration registry that is shared within one process. Write

tests for identity and initialization. Then explain why this does not

synchronize configuration across two server processes.

  

## Exercise B --- Factory

  

Create a notification factory that can produce `EmailNotification`,

`SMSNotification`, and `PushNotification`. Each product should implement

`send(message)`. Unknown types must fail clearly.

  

## Exercise C --- Abstract Factory

  

Create two storage families: - Local storage: local file reader and

local file writer. - Cloud storage: cloud reader and cloud writer.

  

The application should select one family and never manually mix products

from different families. Mock external I/O in tests.

  

## Exercise D --- Builder

  

Build a `DatabaseConfig` with host, port, database name, optional TLS,

connection timeout, and pool size. Validate required fields, port range,

positive timeout, and pool size.

  

## Exercise E --- Design choice

  

For each scenario, justify the simplest appropriate pattern: 1. A report

with many optional formatting parameters. 2. A single application

service used by many components. 3. A system supporting multiple payment

gateways through one payment interface. 4. A group of cloud-specific

storage and authentication components that must work together.

  

For each answer, explain not just why the pattern fits, but why a

simpler design might be insufficient.

  

------------------------------------------------------------------------

  

# 12. Suggested hands-on workflow

  

For each pattern:

  

1.  **Write the naive version first.** Identify duplicated construction,

    invalid states, mismatched products, or unclear arguments.

2.  **State the invariant.** For example: "all connectors come from the

    same exchange family."

3.  **Draw a small class diagram.** Include abstractions, concrete

    classes, and creation methods.

4.  **Implement the smallest version.** Avoid extra layers until a

    requirement justifies them.

5.  **Test the failure paths.** Unknown product type, missing required

    field, invalid configuration, concurrency, or mismatched family.

6.  **Add one new implementation.** Measure how many unrelated modules

    need changes.

7.  **Review complexity.** If the pattern made the design harder to

    follow without solving a recurring problem, simplify it.

  

------------------------------------------------------------------------

  

# 13. Final revision sheet

  

-   **Design pattern:** a reusable design idea, not a universal code

    template.

-   **Creational:** object creation and assembly.

-   **Singleton:** one instance within a defined scope; common access

    point.

-   **Factory:** centralizes selection and creation of a product

    implementation.

-   **Abstract Factory:** creates a compatible family of related product

    types.

-   **Builder:** assembles a complex object step by step using named

    configuration methods.

-   **Prototype:** creates an object by copying an existing object.

-   **Most important contrast:** one product choice versus a family of

    products versus step-by-step assembly.

-   **Production rule:** patterns do not replace validation,

    synchronization, error handling, tests, security, or clear

    dependency management.

-   **Interview rule:** explain the recurring problem first, then the

    pattern, class roles, trade-offs, and a concrete use case.

  

The video's central lesson is that creational patterns help make object

construction more controlled, readable, and extensible. The practical

goal is not to memorize pattern names; it is to recognize when

construction logic is creating coupling, ambiguity, invalid

combinations, or unnecessary complexity---and choose the smallest design

that solves that problem.

  

------------------------------------------------------------------------

  

# 14. Flowchart diagrams

  

The following Mermaid flowcharts can be viewed in Markdown editors that

support Mermaid (for example, GitHub or Obsidian with Mermaid support).

They are also useful as interview-revision diagrams.

  

## 14.1 Choosing a creational pattern

  

``` mermaid

flowchart TD

    A([Start: object creation problem]) --> B{Must one shared instance exist?}

    B -- Yes --> C[Consider Singleton]

    B -- No --> D{Need to choose one product implementation?}

    D -- Yes --> E[Consider Factory]

    D -- No --> F{Need a compatible family of product types?}

    F -- Yes --> G[Consider Abstract Factory]

    F -- No --> H{Many optional fields or complex construction?}

    H -- Yes --> I[Consider Builder]

    H -- No --> J{Need to copy an existing object?}

    J -- Yes --> K[Consider Prototype or a suitable copy operation]

    J -- No --> L[Use a straightforward constructor or function]

```

  

**How to use it:** Treat the chart as a first-pass guide, not a strict

rule. Choose the simplest design that solves the actual recurring

problem.

  

## 14.2 Singleton --- instance access flow

  

``` mermaid

flowchart TD

    A([Caller requests Logger]) --> B{Has instance been created?}

    B -- No --> C[Acquire initialization lock]

    C --> D{Instance still absent?}

    D -- Yes --> E[Create and initialize one instance]

    D -- No --> F[Reuse existing instance]

    E --> G[Return shared instance]

    F --> G

    B -- Yes --> G

    G --> H[Caller writes log message]

```

  

**Implementation note:** This diagram illustrates a thread-safe

first-initialization strategy. Real implementations vary by language.

The logger's write operation may need its own synchronization even if

instance creation is safe.

  

## 14.3 Factory --- selecting one product

  

``` mermaid

flowchart TD

    A([Client needs an order]) --> B[Call OrderFactory.create]

    B --> C{Requested order type}

    C -- market --> D[Create MarketOrder]

    C -- limit --> E[Validate limit price]

    E --> F[Create LimitOrder]

    C -- unknown --> G[Raise clear error]

    D --> H[Return Order abstraction]

    F --> H

    H --> I[Client calls order.execute]

```

  

**Key idea:** The client asks for an order; the factory centralizes the

decision about which concrete order to create.

  

## 14.4 Abstract Factory --- creating a matching product family

  

``` mermaid

flowchart TD

    A([Application starts]) --> B{Which exchange?}

    B -- NYSE --> C[Select NyseFactory]

    B -- NASDAQ --> D[Select NasdaqFactory]

    C --> E[Create NYSE order connector]

    C --> F[Create NYSE market-data connector]

    D --> G[Create NASDAQ order connector]

    D --> H[Create NASDAQ market-data connector]

    E --> I[Use matched NYSE connector family]

    F --> I

    G --> J[Use matched NASDAQ connector family]

    H --> J

    I --> K([Client uses product interfaces])

    J --> K

```

  

**Key idea:** Choose one concrete factory, then create all related

products from that factory. This reduces accidental mixing of

incompatible implementations.

  

## 14.5 Builder --- constructing an HTTP request

  

``` mermaid

flowchart TD

    A([Start]) --> B[Create HttpRequestBuilder]

    B --> C[Set URL]

    C --> D[Set method]

    D --> E[Add optional headers]

    E --> F[Set optional body]

    F --> G[Set timeout or other options]

    G --> H[Call build]

    H --> I{Required fields and rules valid?}

    I -- No --> J[Raise validation error]

    I -- Yes --> K[Create final HttpRequest]

    K --> L[Return completed request]

```

  

**Key idea:** Named methods make each choice explicit; `build()`

validates and returns the final object.

  

## 14.6 Naive constructor versus Builder

  

``` mermaid

flowchart LR

    subgraph Naive["Long positional constructor"]

        A1[Many unnamed values] --> A2[Hard to read]

        A2 --> A3[Same-type values may be swapped]

        A3 --> A4[Errors can survive compilation]

    end

    subgraph Builder["Named builder methods"]

        B1[Set URL] --> B2[Set method]

        B2 --> B3[Set body and options]

        B3 --> B4[Validate with build]

        B4 --> B5[Completed request]

    end

```

  

## 14.7 Pattern comparison at a glance

  

``` mermaid

flowchart TD

    A[Creation responsibility] --> B[Singleton]

    A --> C[Factory]

    A --> D[Abstract Factory]

    A --> E[Builder]

    A --> F[Prototype]

    B --> B1[Control instance count and access]

    C --> C1[Choose one concrete product]

    D --> D1[Create related products as a family]

    E --> E1[Assemble one complex object step by step]

    F --> F1[Copy an existing object]

```

  

## 14.8 Suggested implementation and testing workflow

  

``` mermaid

flowchart TD

    A([Identify recurring problem]) --> B[Write a simple baseline]

    B --> C[State the invariant or failure risk]

    C --> D[Choose the smallest suitable pattern]

    D --> E[Draw class and flow diagrams]

    E --> F[Implement minimal version]

    F --> G[Write success-path tests]

    G --> H[Write failure and edge-case tests]

    H --> I[Add a new implementation or option]

    I --> J{Did the design reduce coupling without excess complexity?}

    J -- Yes --> K[Keep pattern and document trade-offs]

    J -- No --> L[Simplify or reconsider pattern]

```

  

## 14.9 Quick recall diagram

  

-   **Singleton:** one instance in a defined scope.

-   **Factory:** one product abstraction, several possible concrete

    implementations.

-   **Abstract Factory:** several related product types that should be

    compatible.

-   **Builder:** one complex object assembled through named steps.

-   **Prototype:** a new object created by copying an existing one.