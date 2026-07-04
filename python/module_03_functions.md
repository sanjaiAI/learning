# Module 03: Functions
# மாடுல் 03: Functions (செயல்பாடுகள்)

---

## 🎯 What? | என்ன?

**English:** Functions = reusable blocks of code. Master args, kwargs, lambda, closures, and decorators (interview favorite!).

**தமிழ்:** Functions = reusable code blocks. args, kwargs, lambda, closures, decorators master செய்ய வேண்டும் (interview-ல் நிறைய கேட்பார்கள்!).

---

## 🛠️ Function Basics

```python
# Basic function
def greet(name):
    """Greet a person. (This is a docstring)"""
    return f"Hello, {name}!"

result = greet("Sanjai")  # "Hello, Sanjai!"

# Multiple return values (actually returns tuple)
def divide(a, b):
    quotient = a // b
    remainder = a % b
    return quotient, remainder

q, r = divide(17, 5)  # q=3, r=2

# Type hints (documentation, not enforced!)
def add(x: int, y: int) -> int:
    return x + y
```

---

## 🛠️ Arguments | Arguments (INTERVIEW CRITICAL!)

```python
# --- Default arguments ---
def connect(host, port=5432, timeout=30):
    print(f"Connecting to {host}:{port} (timeout={timeout}s)")

connect("db.server.com")              # Uses defaults
connect("db.server.com", port=3306)   # Override port

# ⚠️ TRAP: Mutable default argument!
def append_to(item, lst=[]):    # BUG! Shared across calls!
    lst.append(item)
    return lst

print(append_to(1))  # [1]
print(append_to(2))  # [1, 2] — NOT [2]! Same list!

# ✓ Fix: Use None as default
def append_to(item, lst=None):
    if lst is None:
        lst = []
    lst.append(item)
    return lst

# --- *args (variable positional arguments) ---
def sum_all(*args):
    """Accept any number of arguments."""
    return sum(args)  # args is a tuple

sum_all(1, 2, 3)      # 6
sum_all(1, 2, 3, 4, 5)  # 15

# --- **kwargs (variable keyword arguments) ---
def create_user(**kwargs):
    """Accept any keyword arguments."""
    for key, value in kwargs.items():
        print(f"{key} = {value}")

create_user(name="Sanjai", role="Architect", exp=14)

# --- Combined (order matters!) ---
def func(pos, /, normal, *args, keyword_only, **kwargs):
    pass
# pos         = positional only (before /)
# normal      = positional or keyword
# *args       = extra positional
# keyword_only = must use keyword (after *)
# **kwargs    = extra keyword

# --- Unpacking ---
def add(a, b, c):
    return a + b + c

numbers = [1, 2, 3]
add(*numbers)  # Unpack list as positional args

config = {"a": 1, "b": 2, "c": 3}
add(**config)  # Unpack dict as keyword args
```

---

## 🛠️ Lambda (Anonymous Functions)

```python
# Lambda = one-line function (no name)
square = lambda x: x ** 2
square(5)  # 25

# Common use: sort key
students = [("Alice", 85), ("Bob", 92), ("Charlie", 78)]
students.sort(key=lambda s: s[1], reverse=True)
# [('Bob', 92), ('Alice', 85), ('Charlie', 78)]

# With map, filter
numbers = [1, 2, 3, 4, 5]
doubled = list(map(lambda x: x * 2, numbers))      # [2, 4, 6, 8, 10]
evens = list(filter(lambda x: x % 2 == 0, numbers))  # [2, 4]

# Prefer comprehensions over map/filter (more readable):
doubled = [x * 2 for x in numbers]
evens = [x for x in numbers if x % 2 == 0]
```

---

## 🛠️ Closures | Closures

```python
# Closure = inner function that "remembers" outer function's variables
def make_multiplier(factor):
    def multiply(x):
        return x * factor  # 'factor' captured from outer scope!
    return multiply

double = make_multiplier(2)
triple = make_multiplier(3)

double(5)   # 10
triple(5)   # 15

# Practical: counter
def make_counter():
    count = 0
    def increment():
        nonlocal count  # Modify outer variable
        count += 1
        return count
    return increment

counter = make_counter()
counter()  # 1
counter()  # 2
counter()  # 3
```

---

## 🛠️ Decorators (MOST ASKED IN INTERVIEWS!)

```python
# Decorator = function that wraps another function (adds behavior)
# Pattern: takes function → returns enhanced function

# --- Basic decorator ---
def timer(func):
    """Measure execution time of a function."""
    import time
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(f"{func.__name__} took {end - start:.4f}s")
        return result
    return wrapper

@timer
def slow_function():
    import time
    time.sleep(1)
    return "done"

slow_function()  # "slow_function took 1.0012s"

# @timer is syntactic sugar for:
# slow_function = timer(slow_function)

# --- Decorator with arguments ---
def retry(max_attempts=3, delay=1):
    """Retry a function on failure."""
    import time
    def decorator(func):
        def wrapper(*args, **kwargs):
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts:
                        raise
                    print(f"Attempt {attempt} failed: {e}. Retrying...")
                    time.sleep(delay)
        return wrapper
    return decorator

@retry(max_attempts=3, delay=2)
def fetch_data(url):
    import requests
    return requests.get(url)

# --- Preserve function metadata ---
from functools import wraps

def my_decorator(func):
    @wraps(func)  # Preserves __name__, __doc__
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

# --- Multiple decorators (applied bottom-up!) ---
@timer
@retry(max_attempts=3)
def api_call():
    pass
# Equivalent: timer(retry(max_attempts=3)(api_call))
```

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────┐
│         FUNCTIONS CHEAT SHEET                    │
├──────────────────────────────────────────────────┤
│ ARGUMENT ORDER:                                  │
│   def f(pos, /, normal, *args, kw_only, **kwargs)│
│                                                  │
│ *args   = tuple of extra positional              │
│ **kwargs = dict of extra keyword                 │
│ *list   = unpack list to args                    │
│ **dict  = unpack dict to kwargs                  │
│                                                  │
│ LAMBDA:                                          │
│   lambda x: x * 2  (anonymous, one expression)   │
│   Common: sort key, map/filter                   │
│                                                  │
│ DECORATOR TEMPLATE:                              │
│   def decorator(func):                           │
│       @wraps(func)                               │
│       def wrapper(*args, **kwargs):              │
│           # before                               │
│           result = func(*args, **kwargs)         │
│           # after                                │
│           return result                          │
│       return wrapper                             │
│                                                  │
│ TRAPS:                                           │
│   ❌ def f(lst=[]):  (mutable default!)          │
│   ✓ def f(lst=None): lst = lst or []             │
└──────────────────────────────────────────────────┘
```

---

## 🎤 Interview Q&A | நேர்முகத் தேர்வு

**Q: Write a decorator from scratch.**
```python
from functools import wraps

def log_calls(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__} with {args}, {kwargs}")
        result = func(*args, **kwargs)
        print(f"{func.__name__} returned {result}")
        return result
    return wrapper
```

**Q: What is the mutable default argument trap?**
- Default mutable objects (list, dict) are created ONCE at function definition
- Shared across all calls! Each call modifies the SAME object
- Fix: Use `None` as default, create new object inside function

**Q: *args vs **kwargs?**
- `*args`: collects extra POSITIONAL arguments into a tuple
- `**kwargs`: collects extra KEYWORD arguments into a dict
- Both allow flexible function signatures

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] *args, **kwargs explain and use முடியும்
- [ ] Decorator from scratch write முடியும்
- [ ] Mutable default trap explain முடியும்
- [ ] Closure write முடியும்
- [ ] Lambda with sort/map use முடியும்
