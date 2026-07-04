# Module 05: OOP Fundamentals
# மாடுல் 05: OOP அடிப்படைகள்

---

## 🎯 What? | என்ன?

**English:** Object-Oriented Programming — classes, objects, inheritance, polymorphism, encapsulation. The foundation of Python architecture.

**தமிழ்:** Object-Oriented Programming — classes, objects, inheritance, polymorphism, encapsulation. Python architecture-ன் அடிப்படை.

### Analogy | உதாரணம்
> Class = Blueprint of a house. Object = Actual house built from blueprint. You can build many houses (objects) from one blueprint (class), each with different paint colors (attributes).

---

## 🛠️ Classes & Objects

```python
class Car:
    """A car class."""
    
    # Class variable (shared by ALL instances)
    total_cars = 0
    
    def __init__(self, make, model, year):
        """Constructor — called when creating object."""
        # Instance variables (unique to each object)
        self.make = make
        self.model = model
        self.year = year
        self.speed = 0
        Car.total_cars += 1
    
    def accelerate(self, amount):
        """Increase speed."""
        self.speed += amount
        return self
    
    def brake(self, amount):
        """Decrease speed."""
        self.speed = max(0, self.speed - amount)
        return self
    
    def __str__(self):
        """Human-readable string (print())."""
        return f"{self.year} {self.make} {self.model}"
    
    def __repr__(self):
        """Developer-readable string (debugging)."""
        return f"Car('{self.make}', '{self.model}', {self.year})"


# Create objects
car1 = Car("Toyota", "Camry", 2024)
car2 = Car("Honda", "Civic", 2023)

car1.accelerate(60)
print(car1)          # "2024 Toyota Camry"
print(car1.speed)    # 60
print(Car.total_cars)  # 2

# Method chaining (returns self)
car1.accelerate(20).brake(10)  # speed = 70
```

---

## 🛠️ Inheritance | பரம்பரை

```python
class Vehicle:
    """Base class."""
    def __init__(self, make, model):
        self.make = make
        self.model = model
    
    def start(self):
        return f"{self.make} {self.model} started"
    
    def stop(self):
        return f"{self.make} {self.model} stopped"


class ElectricCar(Vehicle):
    """Child class — inherits from Vehicle."""
    
    def __init__(self, make, model, battery_kwh):
        super().__init__(make, model)  # Call parent __init__
        self.battery_kwh = battery_kwh
        self.charge_level = 100
    
    def charge(self):
        self.charge_level = 100
        return f"{self.model} fully charged"
    
    # Override parent method
    def start(self):
        if self.charge_level > 0:
            return f"{self.model} silently started (electric!)"
        return f"{self.model} cannot start — no charge!"


class HybridCar(Vehicle):
    """Another child class."""
    
    def __init__(self, make, model, battery_kwh, fuel_capacity):
        super().__init__(make, model)
        self.battery_kwh = battery_kwh
        self.fuel_capacity = fuel_capacity


# Usage
tesla = ElectricCar("Tesla", "Model 3", 75)
print(tesla.start())    # "Model 3 silently started (electric!)"
print(tesla.stop())     # "Tesla Model 3 stopped" (inherited!)

# isinstance / issubclass
isinstance(tesla, ElectricCar)  # True
isinstance(tesla, Vehicle)      # True (parent too!)
issubclass(ElectricCar, Vehicle)  # True
```

---

## 🛠️ Polymorphism | பல வடிவம்

```python
# Same method name, different behavior based on object type
class Shape:
    def area(self):
        raise NotImplementedError("Subclasses must implement area()")

class Circle(Shape):
    def __init__(self, radius):
        self.radius = radius
    
    def area(self):
        import math
        return math.pi * self.radius ** 2

class Rectangle(Shape):
    def __init__(self, width, height):
        self.width = width
        self.height = height
    
    def area(self):
        return self.width * self.height

# Polymorphism in action — same interface, different types!
shapes = [Circle(5), Rectangle(3, 4), Circle(2)]
total_area = sum(shape.area() for shape in shapes)
# Each shape.area() calls the RIGHT implementation!
```

---

## 🛠️ Encapsulation | மறைப்பு

```python
class BankAccount:
    def __init__(self, owner, balance=0):
        self.owner = owner          # Public
        self._balance = balance     # Protected (convention: single _)
        self.__pin = "1234"         # Private (name mangling: double __)
    
    @property
    def balance(self):
        """Getter — read-only access."""
        return self._balance
    
    @balance.setter
    def balance(self, amount):
        """Setter — with validation!"""
        if amount < 0:
            raise ValueError("Balance cannot be negative!")
        self._balance = amount
    
    def deposit(self, amount):
        if amount <= 0:
            raise ValueError("Deposit must be positive")
        self._balance += amount
    
    def withdraw(self, amount):
        if amount > self._balance:
            raise ValueError("Insufficient funds")
        self._balance -= amount


account = BankAccount("Sanjai", 1000)
print(account.balance)      # 1000 (uses @property getter)
account.deposit(500)        # 1500
# account.balance = -100    # ValueError! (setter validates)
# account.__pin             # AttributeError! (name mangled)
# account._BankAccount__pin  # "1234" (can still access — Python trusts you)
```

---

## 🛠️ Class Methods & Static Methods

```python
class Employee:
    raise_amount = 1.05  # 5% raise
    employee_count = 0
    
    def __init__(self, name, salary):
        self.name = name
        self.salary = salary
        Employee.employee_count += 1
    
    # Regular method — operates on instance (self)
    def apply_raise(self):
        self.salary *= self.raise_amount
    
    # Class method — operates on CLASS (cls), not instance
    @classmethod
    def set_raise_amount(cls, amount):
        cls.raise_amount = amount
    
    # Alternative constructor (common pattern!)
    @classmethod
    def from_string(cls, emp_str):
        """Create Employee from 'name-salary' string."""
        name, salary = emp_str.split("-")
        return cls(name, int(salary))
    
    # Static method — no access to instance OR class
    @staticmethod
    def is_workday(day):
        """Utility — doesn't need self or cls."""
        return day.weekday() < 5


# Usage
emp1 = Employee("Sanjai", 100000)
emp2 = Employee.from_string("Alice-90000")  # Alternative constructor!

Employee.set_raise_amount(1.10)  # Change for ALL employees
Employee.is_workday(date.today())  # Static utility
```

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────┐
│         OOP FUNDAMENTALS CHEAT SHEET             │
├──────────────────────────────────────────────────┤
│ 4 PILLARS:                                       │
│   Encapsulation  = hide internals (@property)    │
│   Inheritance    = reuse (class Child(Parent))   │
│   Polymorphism   = same interface, diff behavior │
│   Abstraction    = hide complexity               │
│                                                  │
│ METHOD TYPES:                                    │
│   def method(self):    → instance method         │
│   @classmethod         → class method (cls)      │
│   @staticmethod        → utility (no self/cls)   │
│   @property            → getter/setter           │
│                                                  │
│ NAMING:                                          │
│   self.public      → accessible everywhere       │
│   self._protected  → convention: internal use    │
│   self.__private   → name mangled (harder to access)│
│                                                  │
│ MAGIC METHODS:                                   │
│   __init__  = constructor                        │
│   __str__   = print() friendly                   │
│   __repr__  = developer/debug string             │
│   __eq__    = equality comparison                │
│   __len__   = len() support                      │
└──────────────────────────────────────────────────┘
```

---

## 🎤 Interview Q&A | நேர்முகத் தேர்வு

**Q: @classmethod vs @staticmethod?**
- `@classmethod`: receives `cls`, can access/modify class state, used as alternative constructors
- `@staticmethod`: no access to instance or class, pure utility function that belongs logically to the class

**Q: What is `super()`?**
- Calls parent class method. `super().__init__()` calls parent's constructor.
- Follows MRO (Method Resolution Order) for multiple inheritance.

**Q: How does @property work?**
- Makes a method look like an attribute. `account.balance` calls the getter method.
- Allows validation in setter without changing caller's syntax.

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] Class with __init__, __str__, __repr__ write முடியும்
- [ ] Inheritance with super() implement முடியும்
- [ ] @property with getter/setter write முடியும்
- [ ] @classmethod vs @staticmethod explain முடியும்
- [ ] Polymorphism demonstrate முடியும்
