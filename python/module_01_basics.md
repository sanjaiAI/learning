# Module 01: Python Basics
# மாடுல் 01: Python அடிப்படைகள்

---

## 🎯 What? | என்ன?

**English:** Python = readable, powerful, versatile language. Used everywhere — web, DevOps, AI, automation. Simple syntax makes it fast to learn.

**தமிழ்:** Python = readable, powerful language. எல்லா இடத்திலும் use — web, DevOps, AI, automation. Simple syntax = fast to learn.

### Analogy | உதாரணம்
> English among programming languages: Simple grammar, widely understood, you can express complex ideas clearly without verbose syntax.

---

## 🔑 Data Types | தரவு வகைகள்

```python
# --- Numbers ---
x = 10          # int
y = 3.14        # float
z = 2 + 3j      # complex (rarely used)

# --- Strings ---
name = "Sanjai"
greeting = f"Hello, {name}!"         # f-string (formatted)
multiline = """
This is
multiple lines
"""

# --- Boolean ---
is_active = True
is_empty = False

# --- None (null equivalent) ---
result = None

# --- Type checking ---
print(type(x))        # <class 'int'>
print(isinstance(x, int))  # True
```

---

## 🔑 Operators | Operators

```python
# Arithmetic
10 + 3   # 13
10 - 3   # 7
10 * 3   # 30
10 / 3   # 3.333... (float division)
10 // 3  # 3 (integer division — floor)
10 % 3   # 1 (modulo — remainder)
2 ** 10  # 1024 (power)

# Comparison
x == y   # Equal
x != y   # Not equal
x > y    # Greater
x >= y   # Greater or equal

# Logical
True and False   # False
True or False    # True
not True         # False

# Identity (is vs ==)
a = [1, 2, 3]
b = [1, 2, 3]
a == b    # True (same VALUE)
a is b    # False (different OBJECT in memory!)

c = a
a is c    # True (same object!)

# Membership
3 in [1, 2, 3]       # True
"x" not in "hello"   # True
```

---

## 🔑 String Operations | String வேலைகள்

```python
s = "Hello, World!"

# Indexing & Slicing
s[0]        # 'H'
s[-1]       # '!'
s[0:5]      # 'Hello'
s[7:]       # 'World!'
s[::-1]     # '!dlroW ,olleH' (reverse!)

# Methods
s.upper()           # 'HELLO, WORLD!'
s.lower()           # 'hello, world!'
s.strip()           # Remove whitespace from edges
s.split(", ")       # ['Hello', 'World!']
s.replace("World", "Python")  # 'Hello, Python!'
s.startswith("Hello")  # True
s.find("World")        # 7 (index, -1 if not found)

# f-strings (ALWAYS use this for formatting!)
name = "Sanjai"
age = 14  # years of experience
print(f"{name} has {age} years experience")
print(f"Pi = {3.14159:.2f}")  # Pi = 3.14

# String is IMMUTABLE (cannot change in place)
# s[0] = 'h'  # ERROR! TypeError
s = 'h' + s[1:]  # Create new string instead
```

---

## 🔑 Collections Quick Look | Collections

```python
# List (ordered, mutable, duplicates OK)
fruits = ["apple", "banana", "cherry"]
fruits.append("date")
fruits[0]  # "apple"

# Tuple (ordered, IMMUTABLE, duplicates OK)
point = (10, 20)
# point[0] = 5  # ERROR! Cannot modify

# Dictionary (key-value pairs, O(1) lookup)
person = {"name": "Sanjai", "role": "Architect"}
person["name"]  # "Sanjai"
person["age"] = 14  # Add new key

# Set (unordered, unique values only)
unique = {1, 2, 3, 3, 3}  # {1, 2, 3}
```

---

## 🔑 Input/Output | I/O

```python
# Output
print("Hello")
print("Name:", "Sanjai", sep=" | ")  # Name: | Sanjai
print("No newline", end="")          # No newline at end

# Input (interview: rarely needed, but know it)
name = input("Enter name: ")
age = int(input("Enter age: "))  # Convert to int!

# Type conversion
str(42)      # "42"
int("42")    # 42
float("3.14")  # 3.14
list("hello")  # ['h', 'e', 'l', 'l', 'o']
```

---

## 🔑 Variables & Memory | Variables

```python
# Python is dynamically typed (no type declaration needed)
x = 10       # x is int
x = "hello"  # now x is string (no error!)

# Multiple assignment
a, b, c = 1, 2, 3
x = y = z = 0

# Swap (Pythonic!)
a, b = b, a   # No temp variable needed!

# Mutable vs Immutable (CRITICAL for interviews!)
# Immutable: int, float, str, tuple, frozenset
# Mutable: list, dict, set

# This matters for function arguments:
def modify(lst):
    lst.append(4)  # Modifies ORIGINAL list!

my_list = [1, 2, 3]
modify(my_list)
print(my_list)  # [1, 2, 3, 4] — changed!

def modify_str(s):
    s = s + " world"  # Creates NEW string (original unchanged)

my_str = "hello"
modify_str(my_str)
print(my_str)  # "hello" — unchanged!
```

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────┐
│         PYTHON BASICS CHEAT SHEET                │
├──────────────────────────────────────────────────┤
│ TYPES:                                           │
│   int, float, str, bool, None                    │
│   list, tuple, dict, set                         │
│                                                  │
│ MUTABLE vs IMMUTABLE:                            │
│   Mutable: list, dict, set (can change!)         │
│   Immutable: int, str, tuple (cannot change)     │
│                                                  │
│ STRINGS:                                         │
│   f"Hello {name}" (f-string formatting)          │
│   s[start:end:step] (slicing)                    │
│   s.split(), s.join(), s.strip()                 │
│                                                  │
│ OPERATORS:                                       │
│   //  = integer division                         │
│   **  = power                                    │
│   is  = identity (same object?)                  │
│   ==  = equality (same value?)                   │
│   in  = membership                               │
│                                                  │
│ TIPS:                                            │
│   a, b = b, a (swap)                             │
│   x = x or default_value (default)              │
│   isinstance(x, int) (type check)               │
└──────────────────────────────────────────────────┘
```

---

## 🎤 Interview Q&A | நேர்முகத் தேர்வு

**Q: Difference between `is` and `==`?**
- `==` compares VALUES (are they equal?)
- `is` compares IDENTITY (same object in memory?)
- `[1,2] == [1,2]` → True (same value)
- `[1,2] is [1,2]` → False (different objects)

**Q: What are mutable and immutable types?**
- Immutable: int, str, tuple — once created, cannot change. Operations create new objects.
- Mutable: list, dict, set — can modify in place. Be careful passing to functions!

**Q: What does `a, b = b, a` do internally?**
- Python creates a tuple `(b, a)` on the right side, then unpacks to `a` and `b`. No temp variable needed.

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] All data types list முடியும்
- [ ] Mutable vs immutable explain முடியும்
- [ ] String slicing use முடியும்
- [ ] f-string formatting write முடியும்
- [ ] `is` vs `==` difference explain முடியும்
