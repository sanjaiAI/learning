# Python Learning Path — Interview Ready in 2 Weeks
# Python கற்றல் பாதை — 2 வாரத்தில் Interview Ready

> **Level:** Zero → Interview Proficient  
> **Focus:** General Python, DSA, OOP, Design Patterns  
> **Format:** Bilingual (English + Tamil) | Analogies | Code Examples | Interview Q&A  
> **Setup:** PyCharm Professional (Windows) → SSH → Server (203.57.85.108)

---

## 🖥️ Environment Setup | சூழல் அமைப்பு

### Server (Linux — Python Install)

```bash
# SSH into server
ssh root@203.57.85.108

# Install Python 3.11
apt update && apt install -y python3.11 python3.11-venv python3-pip

# Verify
python3.11 --version

# Create project directory
mkdir -p /root/python-learning
cd /root/python-learning

# Create virtual environment
python3.11 -m venv venv
source venv/bin/activate

# Install useful packages
pip install pytest ipython black flake8
```

### Local Machine (Windows — PyCharm)

1. Download **PyCharm Professional** (30-day trial): https://www.jetbrains.com/pycharm/download/
2. Install → Open → New Project
3. **Configure SSH Interpreter:**
   - File → Settings → Project → Python Interpreter
   - Click ⚙️ → Add → SSH Interpreter
   - Host: `203.57.85.108`, Port: `22`, Username: `root`
   - Interpreter path: `/root/python-learning/venv/bin/python3.11`
4. **Set deployment path:** `/root/python-learning`
5. Now: Write code in PyCharm → Executes on server!

### Alternative: VS Code (FREE, also works)
- Install VS Code + "Remote - SSH" extension
- Connect to server → install Python extension on remote
- Same experience, zero cost

---

## 📋 Module Index | 2-Week Plan

| # | Module | Topic | Day |
|---|--------|-------|-----|
| 01 | [Basics](module_01_basics.md) | Syntax, types, operators, I/O | Day 1 |
| 02 | [Control Flow](module_02_control_flow.md) | if/else, loops, comprehensions | Day 1-2 |
| 03 | [Functions](module_03_functions.md) | Args, kwargs, lambda, closures, decorators | Day 2-3 |
| 04 | [Data Structures](module_04_data_structures.md) | List, dict, set, tuple, collections | Day 3-4 |
| 05 | [OOP Fundamentals](module_05_oop.md) | Classes, inheritance, polymorphism, encapsulation | Day 4-5 |
| 06 | [OOP Advanced](module_06_oop_advanced.md) | Magic methods, abstract classes, mixins, SOLID | Day 5-6 |
| 07 | [Error Handling](module_07_error_handling.md) | Exceptions, custom errors, context managers | Day 6 |
| 08 | [File I/O & Serialization](module_08_file_io.md) | Files, JSON, YAML, CSV, pickle | Day 7 |
| 09 | [Iterators & Generators](module_09_iterators_generators.md) | yield, generator expressions, itertools | Day 7-8 |
| 10 | [Design Patterns](module_10_design_patterns.md) | Singleton, Factory, Observer, Strategy, Decorator | Day 8-9 |
| 11 | [DSA — Arrays & Strings](module_11_dsa_arrays.md) | Two pointer, sliding window, sorting | Day 9-10 |
| 12 | [DSA — Linked Lists & Stacks](module_12_dsa_linked_lists.md) | Implementation, stack, queue, monotonic stack | Day 10-11 |
| 13 | [DSA — Trees & Graphs](module_13_dsa_trees.md) | BST, DFS, BFS, traversals | Day 11-12 |
| 14 | [DSA — Hash Maps & Algorithms](module_14_dsa_hashmaps.md) | Hashing, recursion, DP basics | Day 12-13 |
| 15 | [Testing & Best Practices](module_15_testing.md) | pytest, TDD, type hints, clean code | Day 13-14 |

---

## 📅 Daily Schedule | தினசரி திட்டம்

| Time | Activity |
|------|----------|
| Morning (2h) | Read module + understand concepts |
| Afternoon (2h) | Code practice (type every example!) |
| Evening (1h) | Solve 2-3 problems + interview Q&A review |

---

## 🎯 Interview Expectations (General Python)

| Area | What they ask | Weight |
|------|--------------|--------|
| **OOP** | Design a class hierarchy, SOLID principles | 30% |
| **DSA** | Solve coding problem (arrays, strings, trees) | 30% |
| **Language features** | Decorators, generators, context managers | 20% |
| **Design Patterns** | When to use Factory, Singleton, Strategy | 10% |
| **Best Practices** | Testing, error handling, clean code | 10% |

---

## 🔑 Key Python Interview Topics (Must Know)

```
✓ Mutable vs Immutable (list vs tuple)
✓ List comprehension vs generator expression
✓ *args, **kwargs
✓ Decorators (write one from scratch!)
✓ @property, @staticmethod, @classmethod
✓ __init__, __repr__, __str__, __eq__
✓ Abstract classes (ABC)
✓ Context managers (with statement)
✓ Generator (yield) vs return
✓ GIL (Global Interpreter Lock)
✓ Deep copy vs shallow copy
✓ Dictionary — O(1) lookup
✓ Two pointer / sliding window patterns
✓ Stack/Queue implementations
✓ Binary search
✓ Basic recursion + memoization
```
