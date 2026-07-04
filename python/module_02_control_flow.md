# Module 02: Control Flow
# மாடுல் 02: Control Flow (நிரல் ஓட்டம்)

---

## 🎯 What? | என்ன?

**English:** Decision making (if/elif/else), repetition (for/while loops), and the Pythonic way to write loops (comprehensions).

**தமிழ்:** Decision making (if/elif/else), repetition (for/while loops), Pythonic loops (comprehensions).

---

## 🛠️ Conditionals | நிபந்தனைகள்

```python
# Basic if/elif/else
age = 25
if age < 18:
    print("Minor")
elif age < 60:
    print("Adult")
else:
    print("Senior")

# Ternary (one-line if)
status = "Adult" if age >= 18 else "Minor"

# Truthy / Falsy (INTERVIEW FAVORITE!)
# Falsy values: None, 0, 0.0, "", [], {}, set(), False
# Everything else is Truthy

if []:          # False (empty list)
    print("won't print")
if [1, 2, 3]:  # True (non-empty list)
    print("will print")

# Pythonic None check
if result is None:      # ✓ Correct
    pass
# if result == None:    # ✗ Works but not Pythonic

# Multiple conditions
if 18 <= age <= 60:     # Python allows chaining!
    print("Working age")

# Match-case (Python 3.10+ — like switch)
command = "start"
match command:
    case "start":
        print("Starting...")
    case "stop":
        print("Stopping...")
    case _:
        print("Unknown command")
```

---

## 🛠️ For Loops | For Loops

```python
# Iterate over list
fruits = ["apple", "banana", "cherry"]
for fruit in fruits:
    print(fruit)

# With index (enumerate — Pythonic!)
for i, fruit in enumerate(fruits):
    print(f"{i}: {fruit}")
# 0: apple
# 1: banana
# 2: cherry

# Range
for i in range(5):       # 0, 1, 2, 3, 4
    print(i)
for i in range(2, 10, 2):  # 2, 4, 6, 8 (start, stop, step)
    print(i)

# Iterate over dict
person = {"name": "Sanjai", "role": "Architect", "exp": 14}
for key, value in person.items():
    print(f"{key}: {value}")

# Zip (parallel iteration)
names = ["Alice", "Bob", "Charlie"]
scores = [85, 92, 78]
for name, score in zip(names, scores):
    print(f"{name}: {score}")

# break, continue, else
for num in range(100):
    if num == 5:
        break       # Exit loop
    if num % 2 == 0:
        continue    # Skip to next iteration
    print(num)      # Prints: 1, 3

# for-else (runs if loop completes WITHOUT break)
for n in range(2, 10):
    for i in range(2, n):
        if n % i == 0:
            break
    else:
        print(f"{n} is prime")  # Only if inner loop didn't break
```

---

## 🛠️ While Loops | While Loops

```python
# Basic while
count = 0
while count < 5:
    print(count)
    count += 1

# Infinite loop with break
while True:
    user_input = input("Enter 'quit' to exit: ")
    if user_input == "quit":
        break

# While with walrus operator (Python 3.8+)
while (line := input("Enter: ")) != "quit":
    print(f"You said: {line}")
```

---

## 🛠️ Comprehensions (VERY IMPORTANT!) | Comprehensions

```python
# --- List Comprehension ---
# Traditional way:
squares = []
for x in range(10):
    squares.append(x ** 2)

# Pythonic way (comprehension):
squares = [x ** 2 for x in range(10)]
# [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]

# With condition (filter)
evens = [x for x in range(20) if x % 2 == 0]
# [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]

# With transformation + condition
words = ["hello", "WORLD", "Python", "is", "GREAT"]
lower_long = [w.lower() for w in words if len(w) > 3]
# ['hello', 'world', 'python', 'great']

# Nested comprehension (flatten 2D list)
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flat = [num for row in matrix for num in row]
# [1, 2, 3, 4, 5, 6, 7, 8, 9]

# --- Dict Comprehension ---
names = ["alice", "bob", "charlie"]
name_lengths = {name: len(name) for name in names}
# {'alice': 5, 'bob': 3, 'charlie': 7}

# Swap keys and values
original = {"a": 1, "b": 2, "c": 3}
swapped = {v: k for k, v in original.items()}
# {1: 'a', 2: 'b', 3: 'c'}

# --- Set Comprehension ---
sentence = "hello world hello python world"
unique_words = {word for word in sentence.split()}
# {'hello', 'world', 'python'}

# --- Generator Expression (lazy — doesn't store all in memory) ---
sum_of_squares = sum(x ** 2 for x in range(1000000))
# Computes on-the-fly, no list in memory!
```

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────┐
│       CONTROL FLOW CHEAT SHEET                   │
├──────────────────────────────────────────────────┤
│ FALSY VALUES:                                    │
│   None, 0, 0.0, "", [], {}, set(), False         │
│   Everything else is Truthy!                     │
│                                                  │
│ LOOPS:                                           │
│   for x in iterable:       (iterate)             │
│   for i, x in enumerate(): (with index)          │
│   for a, b in zip(X, Y):  (parallel)            │
│   while condition:         (repeat until)        │
│                                                  │
│ COMPREHENSIONS:                                  │
│   [expr for x in iter]              (list)       │
│   [expr for x in iter if cond]      (filtered)   │
│   {k: v for k, v in iter}           (dict)       │
│   {expr for x in iter}              (set)        │
│   (expr for x in iter)              (generator)  │
│                                                  │
│ TIPS:                                            │
│   Prefer comprehension over loop+append          │
│   Use enumerate() not range(len())               │
│   Use zip() for parallel iteration               │
│   for-else: else runs if NO break                │
└──────────────────────────────────────────────────┘
```

---

## 🎤 Interview Q&A | நேர்முகத் தேர்வு

**Q: List comprehension vs generator expression?**
- List `[x for x in range(1M)]` = creates full list in memory (1M items stored)
- Generator `(x for x in range(1M))` = lazy, computes one at a time, O(1) memory
- Use generator when you don't need all items at once (sum, any, all)

**Q: What are falsy values in Python?**
- `None`, `0`, `0.0`, `""`, `[]`, `{}`, `set()`, `False`
- Common pattern: `if not my_list:` to check empty

**Q: What does `for-else` do?**
- `else` block runs only if loop completed normally (no `break`)
- Useful for search: if found → break, else → "not found"

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] Truthy/Falsy values list முடியும்
- [ ] List comprehension with filter write முடியும்
- [ ] Dict comprehension write முடியும்
- [ ] enumerate, zip use முடியும்
- [ ] for-else explain முடியும்
