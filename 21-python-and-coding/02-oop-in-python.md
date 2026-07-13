# Object-Oriented Programming in Python

Object-Oriented Programming (OOP) organises code around **objects** — bundles of data and behaviour — rather than loose functions and variables. The four pillars (Encapsulation, Abstraction, Inheritance, Polymorphism) are staple interview topics; this chapter explains each one simply, with analogies and short runnable examples that build on a single `BankAccount` / `Shape` theme.

## Table of Contents

- [Classes & Objects](#classes--objects)
- [Encapsulation](#encapsulation)
- [Abstraction](#abstraction)
- [Inheritance](#inheritance)
- [Polymorphism](#polymorphism)
- [Dunder / Magic Methods](#dunder--magic-methods)
- [Method Types — static, class, property](#method-types----static-class-property)
- [dataclass](#dataclass)
- [Composition vs Inheritance](#composition-vs-inheritance)
- [__slots__](#__slots__)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Classes & Objects

**In plain English:** A **class** is a blueprint (think: a cookie-cutter). An **object** is one specific cookie made from that cutter. Every object gets its own copy of the data defined in `__init__`.

| Term | Meaning |
|---|---|
| `__init__` | The constructor — called automatically when you create an object |
| `self` | Reference to *this particular* object |
| Instance attribute | Stored on the object (`self.balance`) — each object has its own |
| Class attribute | Stored on the class (`BankAccount.interest_rate`) — shared by all objects |

```python
class BankAccount:
    interest_rate = 0.03          # class attribute — shared by all accounts

    def __init__(self, owner: str, balance: float = 0.0):
        self.owner = owner        # instance attribute
        self.balance = balance    # instance attribute

    def deposit(self, amount: float):
        self.balance += amount
        return self.balance

# Create two objects from the same blueprint
alice = BankAccount("Alice", 1000)
bob   = BankAccount("Bob",   500)

print(alice.balance)       # 1000   — alice's own copy
print(BankAccount.interest_rate)  # 0.03 — shared class attribute
alice.deposit(200)
print(alice.balance)       # 1200
```

---

## Encapsulation

**Analogy:** An ATM keeps the cash drawer locked. You interact through a controlled interface (buttons) — you never reach inside the machine directly.

Encapsulation bundles data + behaviour in one unit and **controls access** via naming conventions:

| Convention | Meaning |
|---|---|
| `balance` (no underscore) | Public — anyone can read/write |
| `_balance` (single underscore) | Protected — "please don't touch from outside", but not enforced |
| `__balance` (double underscore) | Private — Python applies **name mangling**: `_BankAccount__balance` |

Use `@property` to expose controlled access (read + optional validation) without revealing internals.

```python
class BankAccount:
    interest_rate = 0.03

    def __init__(self, owner: str, balance: float = 0.0):
        self.owner = owner
        self.__balance = balance   # ← private; name-mangled to _BankAccount__balance

    @property
    def balance(self):             # getter — read like an attribute
        return self.__balance

    @balance.setter
    def balance(self, amount: float):   # setter — validate before writing
        if amount < 0:
            raise ValueError("Balance cannot be negative")
        self.__balance = amount

    def deposit(self, amount: float):
        if amount <= 0:
            raise ValueError("Deposit must be positive")
        self.__balance += amount

    def __str__(self):
        return f"BankAccount(owner={self.owner!r}, balance={self.__balance:.2f})"

acc = BankAccount("Alice", 1000)
acc.deposit(200)
print(acc.balance)     # 1200.0  — via property getter
# acc.__balance        # AttributeError — can't access name-mangled attribute directly
```

---

## Abstraction

**Analogy:** A car's steering wheel hides the engine, gearbox, and hydraulics. You just turn the wheel — you don't need to know *how* it works.

Abstraction defines **what** a class must do without dictating **how**. In Python, use `abc.ABC` + `@abstractmethod`:

```python
from abc import ABC, abstractmethod

class Shape(ABC):                  # abstract blueprint — cannot be instantiated
    @abstractmethod
    def area(self) -> float:       # every Shape MUST implement this
        ...

    @abstractmethod
    def perimeter(self) -> float:
        ...

    def describe(self):            # concrete method — shared implementation
        return f"Area={self.area():.2f}, Perimeter={self.perimeter():.2f}"


class Circle(Shape):
    def __init__(self, radius: float):
        self.radius = radius

    def area(self):
        import math
        return math.pi * self.radius ** 2

    def perimeter(self):
        import math
        return 2 * math.pi * self.radius


class Rectangle(Shape):
    def __init__(self, w: float, h: float):
        self.w, self.h = w, h

    def area(self):
        return self.w * self.h

    def perimeter(self):
        return 2 * (self.w + self.h)


# Shape()  # → TypeError: Can't instantiate abstract class
c = Circle(5)
print(c.describe())          # Area=78.54, Perimeter=31.42
print(Rectangle(4, 6).describe())  # Area=24.00, Perimeter=20.00
```

> **Key interview point:** If a subclass does not implement all `@abstractmethod`s it also becomes abstract and cannot be instantiated (raises `TypeError`).

---

## Inheritance

**Analogy:** A `SavingsAccount` *is a* `BankAccount` with extra rules (e.g., minimum balance). It inherits everything from the parent and adds or overrides what's different.

```python
class BankAccount:
    def __init__(self, owner: str, balance: float = 0.0):
        self.owner = owner
        self._balance = balance

    def deposit(self, amount: float):
        self._balance += amount

    def __str__(self):
        return f"{type(self).__name__}({self.owner}, ${self._balance:.2f})"


class SavingsAccount(BankAccount):
    MINIMUM_BALANCE = 100.0

    def __init__(self, owner: str, balance: float = 0.0, interest_rate: float = 0.05):
        super().__init__(owner, balance)   # call parent __init__
        self.interest_rate = interest_rate

    def apply_interest(self):
        self._balance += self._balance * self.interest_rate

    def withdraw(self, amount: float):
        if self._balance - amount < self.MINIMUM_BALANCE:
            raise ValueError(f"Must keep minimum balance of ${self.MINIMUM_BALANCE}")
        self._balance -= amount


sa = SavingsAccount("Bob", 500, interest_rate=0.04)
sa.deposit(100)          # inherited from BankAccount
sa.apply_interest()      # SavingsAccount-specific
print(sa)                # SavingsAccount(Bob, $624.00)
```

### Method Resolution Order (MRO)

Python uses **C3 linearisation** to resolve which class's method is called when multiple inheritance is involved. Use `ClassName.__mro__` or `ClassName.mro()` to inspect.

```python
class A:
    def hello(self): return "A"

class B(A):
    def hello(self): return "B"

class C(A):
    def hello(self): return "C"

class D(B, C):   # multiple inheritance
    pass

print(D.__mro__)   # (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)
print(D().hello()) # "B"  — leftmost parent wins
```

---

## Polymorphism

**Analogy:** A universal TV remote's "volume up" button works whether the TV is a Sony, Samsung, or LG. Same button, different internal implementation.

Polymorphism means different objects respond to the same method name in their own way. Python achieves this naturally via **duck typing** ("if it quacks like a duck…") — no `interface` keyword needed.

```python
def print_shape_info(shape: Shape):   # works for ANY Shape subclass
    print(shape.describe())

shapes = [Circle(3), Rectangle(4, 5), Circle(10)]
for s in shapes:
    print_shape_info(s)   # each calls its own area() / perimeter()
```

### Duck Typing

```python
class Duck:
    def speak(self): return "Quack!"

class Dog:
    def speak(self): return "Woof!"

class Robot:
    def speak(self): return "Beep boop."

# No shared base class needed — just the same method name
for animal in [Duck(), Dog(), Robot()]:
    print(animal.speak())
```

### Method Overriding

A subclass re-defines a parent method. The most derived version always wins.

```python
class BankAccount:
    def transaction_fee(self): return 5.0

class PremiumAccount(BankAccount):
    def transaction_fee(self): return 0.0   # override — waived for premium

accounts = [BankAccount(), PremiumAccount()]
for acc in accounts:
    print(acc.transaction_fee())   # 5.0, then 0.0
```

---

## Dunder / Magic Methods

**In plain English:** Dunder methods (double-underscore on both sides) let your objects plug into Python's built-in syntax — `print()`, `+`, `==`, `len()`, calling like a function, etc.

```python
from dataclasses import dataclass

class Money:
    def __init__(self, amount: float, currency: str = "USD"):
        self.amount = amount
        self.currency = currency

    # --- String representations ---
    def __repr__(self):
        # For developers: unambiguous, should recreate the object if possible
        return f"Money({self.amount!r}, {self.currency!r})"

    def __str__(self):
        # For users: readable
        return f"{self.currency} {self.amount:.2f}"

    # --- Equality ---
    def __eq__(self, other):
        if not isinstance(other, Money):
            return NotImplemented
        return self.amount == other.amount and self.currency == other.currency

    # --- Operator overloading ---
    def __add__(self, other):
        if self.currency != other.currency:
            raise ValueError("Cannot add different currencies")
        return Money(self.amount + other.amount, self.currency)

    # --- Sizing ---
    def __len__(self):
        # Contrived, but shows how len() works
        return int(self.amount)

    # --- Callable ---
    def __call__(self, tax_rate: float):
        """Apply a tax rate and return the total."""
        return Money(self.amount * (1 + tax_rate), self.currency)


m1 = Money(100, "USD")
m2 = Money(50, "USD")

print(str(m1))           # USD 100.00      ← __str__
print(repr(m1))          # Money(100, 'USD') ← __repr__
print(m1 == m2)          # False            ← __eq__
print(m1 + m2)           # USD 150.00       ← __add__ then __str__
print(len(m1))           # 100              ← __len__
print(m1(0.1))           # USD 110.00       ← __call__
```

| Dunder | Triggered by |
|---|---|
| `__init__` | `ClassName(...)` |
| `__str__` | `str(obj)`, `print(obj)` |
| `__repr__` | `repr(obj)`, interactive console |
| `__eq__` | `obj1 == obj2` |
| `__lt__` / `__gt__` | `<`, `>` (implement for sorting) |
| `__add__` | `obj1 + obj2` |
| `__len__` | `len(obj)` |
| `__call__` | `obj(...)` |
| `__enter__` / `__exit__` | `with obj:` (context manager) |
| `__iter__` / `__next__` | `for x in obj:` |

---

## Method Types — static, class, property

```python
class BankAccount:
    _total_accounts = 0          # class-level counter

    def __init__(self, owner: str, balance: float = 0.0):
        self.owner = owner
        self.__balance = balance
        BankAccount._total_accounts += 1

    # --- Instance method (the default) ---
    # Receives `self`; operates on the specific object
    def deposit(self, amount: float):
        self.__balance += amount

    # --- @classmethod ---
    # Receives `cls`; operates on the class itself, not an instance.
    # Use for: alternative constructors, factory methods, accessing class state.
    @classmethod
    def total_accounts(cls):
        return cls._total_accounts

    @classmethod
    def from_dict(cls, data: dict):
        """Alternative constructor — factory pattern."""
        return cls(data["owner"], data.get("balance", 0))

    # --- @staticmethod ---
    # No `self` or `cls`; just a plain function namespaced inside the class.
    # Use for: utility helpers that relate to the class concept but need no class/instance data.
    @staticmethod
    def validate_amount(amount: float) -> bool:
        return amount > 0

    # --- @property ---
    # Turns a method into a read-only attribute; pair with .setter for controlled writes.
    @property
    def balance(self):
        return self.__balance

    @balance.setter
    def balance(self, value: float):
        if value < 0:
            raise ValueError("Balance cannot be negative")
        self.__balance = value


a1 = BankAccount("Alice", 500)
a2 = BankAccount.from_dict({"owner": "Bob", "balance": 200})

print(BankAccount.total_accounts())      # 2   ← classmethod
print(BankAccount.validate_amount(-5))  # False ← staticmethod
print(a1.balance)                        # 500  ← property getter
a1.balance = 600                         # property setter
```

| Decorator | First param | Typical use |
|---|---|---|
| *(none)* | `self` | Normal instance methods |
| `@classmethod` | `cls` | Factory methods, class-wide state |
| `@staticmethod` | *(none)* | Utility functions loosely grouped with the class |
| `@property` | `self` | Controlled attribute access with validation |

---

## dataclass

**In plain English:** `@dataclass` is syntactic sugar — it auto-generates `__init__`, `__repr__`, and `__eq__` so you don't have to write boilerplate.

```python
from dataclasses import dataclass, field

@dataclass
class Transaction:
    amount: float
    description: str
    tags: list[str] = field(default_factory=list)   # mutable default must use field()

    def __post_init__(self):
        if self.amount <= 0:
            raise ValueError("Transaction amount must be positive")


t1 = Transaction(50.0, "Coffee")
t2 = Transaction(50.0, "Coffee")

print(t1)          # Transaction(amount=50.0, description='Coffee', tags=[])
print(t1 == t2)    # True  — __eq__ auto-generated
```

Use `@dataclass(frozen=True)` to make instances immutable (hashable, usable in sets/dict keys).  
Use `@dataclass(slots=True)` (Python 3.10+) for memory efficiency (same as `__slots__` below).

---

## Composition vs Inheritance

> **Rule of thumb:** Prefer **composition** ("has-a") over inheritance ("is-a") when the relationship is one of *using* rather than *being*.

- **Inheritance** is right when `SavingsAccount` truly *is a* `BankAccount`.
- **Composition** is right when a `BankAccount` *has an* `AuditLogger` — it uses it, it's not a subclass of it.

```python
class AuditLogger:
    def __init__(self):
        self._log: list[str] = []

    def record(self, message: str):
        self._log.append(message)

    def history(self):
        return list(self._log)


class BankAccount:
    def __init__(self, owner: str):
        self.owner = owner
        self._balance = 0.0
        self._logger = AuditLogger()    # ← composition: BankAccount HAS an AuditLogger

    def deposit(self, amount: float):
        self._balance += amount
        self._logger.record(f"Deposit ${amount:.2f}")

    def withdraw(self, amount: float):
        self._balance -= amount
        self._logger.record(f"Withdrawal ${amount:.2f}")

    def history(self):
        return self._logger.history()


acc = BankAccount("Alice")
acc.deposit(500)
acc.withdraw(100)
print(acc.history())   # ['Deposit $500.00', 'Withdrawal $100.00']
```

**Why prefer composition?**
- Avoids deep, fragile inheritance hierarchies.
- Each component is independently testable.
- Behaviour can be swapped at runtime (e.g., swap `AuditLogger` for a `CloudLogger`).

---

## __slots__

**In plain English:** By default Python stores object attributes in a hidden `__dict__` (a regular Python dict), which has memory overhead. `__slots__` pre-declares allowed attribute names so Python uses a compact fixed-size structure instead — typically **20–40 % less memory** per object, useful when creating millions of instances.

```python
class Point:
    __slots__ = ("x", "y")     # ONLY these attributes are allowed

    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y


p = Point(1.0, 2.0)
print(p.x, p.y)     # 1.0 2.0
# p.z = 3.0         # AttributeError — not in __slots__
# p.__dict__        # AttributeError — no __dict__ when __slots__ is used
```

> Caveat: `__slots__` classes can't be pickled as easily and don't play well with multiple inheritance unless all bases also define `__slots__`.

---

## Interview Questions

### Q: What are the four pillars of OOP and how does Python implement each?

**Strong answer:**
**Encapsulation** — bundle data + methods, control access with `_`/`__` naming and `@property`. **Abstraction** — use `abc.ABC` + `@abstractmethod` to define contracts without implementation. **Inheritance** — `class Child(Parent)` reuses/extends parent; `super()` calls the parent; MRO governs lookup order. **Polymorphism** — duck typing + method overriding let different objects respond to the same interface without a shared base class requirement.

---

### Q: What is name mangling in Python?

**Strong answer:**
When you prefix an attribute with `__` (double underscore) inside a class, Python renames it to `_ClassName__attr` at compile time. This makes accidental overrides in subclasses harder, but it is **not** true private access — you can still reach `obj._ClassName__attr` if you know the mangled name. It's a convention, not a security guarantee.

---

### Q: What is the difference between `__str__` and `__repr__`?

**Strong answer:**
`__repr__` is for **developers** — it should return an unambiguous string that ideally lets you recreate the object (e.g., `Money(100, 'USD')`). `__str__` is for **end-users** — it should be readable (e.g., `USD 100.00`). `print()` calls `__str__`; the interactive console and `repr()` call `__repr__`. If only `__repr__` is defined, `str()` falls back to it — so always define `__repr__` at minimum.

---

### Q: When would you use `@classmethod` vs `@staticmethod`?

**Strong answer:**
Use `@classmethod` when the method needs access to the **class itself** — the most common case is an alternative constructor (factory method) like `BankAccount.from_dict(data)`. Use `@staticmethod` when the function is logically related to the class but needs **neither `self` nor `cls`** — pure utility helpers like `BankAccount.validate_amount(50)`. If you need the specific object's state, use a plain instance method.

---

### Q: Why can't you instantiate an abstract class?

**Strong answer:**
`abc.ABCMeta` (applied via `ABC`) checks at instantiation time that all `@abstractmethod`s have been overridden. If any are missing, Python raises `TypeError: Can't instantiate abstract class`. This enforces a contract — every concrete subclass *must* provide the required interface before it can be used. The alternative (`raise NotImplementedError`) only fails at call time, allowing partially-broken objects to exist.

---

### Q: What is the Method Resolution Order (MRO) and why does it matter?

**Strong answer:**
MRO is the order Python searches base classes when looking up a method. Python uses the **C3 linearisation** algorithm, which guarantees a consistent, predictable order even with multiple inheritance. You can inspect it with `ClassName.__mro__`. It matters because in diamond inheritance (`D` inherits from `B` and `C`, both inherit from `A`), C3 ensures `A.__init__` is only called once and the correct override wins.

---

### Q: What is duck typing?

**Strong answer:**
Duck typing means Python cares about **what an object can do**, not what type it is — "if it walks like a duck and quacks like a duck, it's a duck." A function that calls `.area()` works with any object that has that method, regardless of class hierarchy. This enables polymorphism without explicit interfaces or type declarations and is a core reason Python code is flexible and composable.

---

### Q: When should you prefer composition over inheritance?

**Strong answer:**
Prefer composition when the relationship is "**has-a**" rather than "**is-a**". `BankAccount` *has an* `AuditLogger` — composition. `SavingsAccount` *is a* `BankAccount` — inheritance. Deep inheritance hierarchies become brittle: changing a parent breaks all children. Composed objects are independently testable and swappable, making the codebase easier to maintain and extend.

---

### Q: What does `@dataclass` give you automatically?

**Strong answer:**
`@dataclass` auto-generates `__init__` (from the annotated fields), `__repr__`, and `__eq__` (field-by-field comparison). With flags: `frozen=True` adds immutability + `__hash__`; `order=True` adds `__lt__`/`__gt__` for sorting; `slots=True` (Python 3.10+) adds `__slots__` for memory efficiency. Use `field(default_factory=...)` for mutable defaults like lists. It's ideal for plain data-holding classes.

---

### Q: What are `__slots__` and when should you use them?

**Strong answer:**
`__slots__ = ("x", "y")` tells Python to store instance attributes in a fixed-size C struct instead of a per-object `__dict__`. This saves roughly 20–40 % memory per instance and speeds up attribute access. Use `__slots__` when you create **thousands or millions** of small objects (e.g., coordinate points, event records). Trade-offs: no dynamic attribute assignment, harder pickling, and inheritance requires all bases to also define `__slots__`.

---

### Q: What does `super()` do and why is it better than calling the parent class directly?

**Strong answer:**
`super()` returns a proxy object that delegates method calls to the next class in the MRO. Using `super().__init__(...)` instead of `ParentClass.__init__(self, ...)` means the call respects the MRO and cooperates correctly with multiple inheritance — each class in the chain calls the next one once. Hard-coding the parent name breaks this when the hierarchy changes.

---

### Q: What is the difference between a class attribute and an instance attribute?

**Strong answer:**
A **class attribute** is defined in the class body (outside `__init__`) and is shared among all instances — changing it on the class affects every instance that hasn't overridden it. An **instance attribute** is defined inside `__init__` on `self` and belongs to that specific object alone. A common gotcha: if you assign to `obj.class_attr`, Python creates a new *instance* attribute that shadows the class one — the class attribute is unchanged.

---

### Q: Can a `@property` replace all uses of getters and setters?

**Strong answer:**
Yes, in Python `@property` is the idiomatic way to add controlled access to an attribute. You start with a plain attribute, and when you need validation or computation, you refactor to `@property` without changing the callers (they still use `obj.balance`, not `obj.get_balance()`). The `.setter` decorator adds write logic. This keeps the public API clean while hiding internals — exactly what encapsulation is for.

---

## References

- [Python OOP Interview Questions — PYnative](https://pynative.com/python-oop-interview-questions/)
- [Abstract Base Classes (abc) — Python Official Docs](https://docs.python.org/3/library/abc.html)
- [Dunder / Magic Methods in Python — GeeksforGeeks](https://www.geeksforgeeks.org/python/dunder-magic-methods-python/)
- [Data Classes in Python (Guide) — Real Python](https://realpython.com/python-data-classes/)
- [Python OOP Interview Questions 2025: Inheritance, Polymorphism, and Tricky Examples — Medium / CodeToDeploy](https://medium.com/codetodeploy/python-oop-interview-questions-2025-inheritance-polymorphism-and-10-tricky-examples-explained-e71058f84144)

---

*Previous: [Python Core Concepts](01-python-core-concepts.md)*
