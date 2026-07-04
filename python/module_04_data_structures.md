# Module 04: Data Structures
# மாடுல் 04: Data Structures (தரவு கட்டமைப்புகள்)

---

## 🎯 What? | என்ன?

**English:** Master Python's built-in data structures — list, dict, set, tuple + collections module (deque, defaultdict, Counter, OrderedDict).

**தமிழ்:** Python built-in data structures master — list, dict, set, tuple + collections module.

---

## 🛠️ List (Dynamic Array)

```python
# Create
nums = [1, 2, 3, 4, 5]
mixed = [1, "hello", True, 3.14]  # Any type

# Operations — O(1)
nums[0]           # Access by index: O(1)
nums[-1]          # Last element: O(1)
nums.append(6)    # Add to end: O(1)
nums.pop()        # Remove from end: O(1)

# Operations — O(n)
nums.insert(0, 0)  # Insert at index: O(n) — shifts everything!
nums.pop(0)        # Remove from front: O(n)
nums.remove(3)     # Remove first occurrence: O(n)
3 in nums          # Search: O(n)

# Slicing
nums[1:4]         # [2, 3, 4] — new list
nums[::2]         # [1, 3, 5] — every 2nd
nums[::-1]        # [5, 4, 3, 2, 1] — reversed

# Sorting
nums.sort()                  # In-place, returns None
sorted_nums = sorted(nums)   # Returns new list
nums.sort(key=lambda x: -x)  # Sort descending
nums.sort(key=abs)           # Sort by absolute value

# Common patterns
# Remove duplicates (preserve order)
seen = set()
unique = [x for x in nums if x not in seen and not seen.add(x)]

# Flatten nested list
nested = [[1, 2], [3, 4], [5, 6]]
flat = [x for sub in nested for x in sub]  # [1, 2, 3, 4, 5, 6]
```

---

## 🛠️ Dictionary (Hash Map)

```python
# Create
person = {"name": "Sanjai", "role": "Architect", "exp": 14}

# Access
person["name"]           # "Sanjai" — KeyError if missing!
person.get("name")       # "Sanjai" — None if missing (safe)
person.get("age", 0)     # 0 (default value)

# Modify
person["age"] = 30       # Add/update
del person["age"]        # Delete key
person.pop("exp")        # Remove and return value

# Iterate
for key in person:                    # Keys
for key, value in person.items():     # Key-value pairs
for value in person.values():         # Values only

# Useful methods
person.keys()      # dict_keys(['name', 'role'])
person.values()    # dict_values(['Sanjai', 'Architect'])
person.update({"city": "Bangalore", "exp": 14})  # Merge

# Merge (Python 3.9+)
merged = dict1 | dict2   # dict2 values override dict1

# Dict comprehension
squares = {x: x**2 for x in range(1, 6)}
# {1: 1, 2: 4, 3: 9, 4: 16, 5: 25}

# ⚠️ Time complexity: ALL O(1) average!
# Access, insert, delete, lookup = O(1)
```

---

## 🛠️ Set (Unique Elements)

```python
# Create
s = {1, 2, 3, 4, 5}
s = set([1, 1, 2, 2, 3])  # {1, 2, 3} — duplicates removed!

# Operations — O(1)
s.add(6)       # Add element
s.remove(3)    # Remove (KeyError if missing)
s.discard(3)   # Remove (NO error if missing)
3 in s         # Membership check: O(1)!

# Set operations (VERY useful for interviews!)
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}

a | b    # Union: {1, 2, 3, 4, 5, 6}
a & b    # Intersection: {3, 4}
a - b    # Difference: {1, 2}
a ^ b    # Symmetric diff: {1, 2, 5, 6}

# Subset/superset
{1, 2} <= {1, 2, 3}   # True (subset)
{1, 2, 3} >= {1, 2}   # True (superset)

# Common pattern: find duplicates
nums = [1, 2, 3, 2, 4, 3, 5]
seen = set()
duplicates = set()
for n in nums:
    if n in seen:
        duplicates.add(n)
    seen.add(n)
# duplicates = {2, 3}
```

---

## 🛠️ Tuple (Immutable List)

```python
# Create
point = (10, 20)
single = (42,)    # Single element — comma needed!
coords = tuple([1, 2, 3])

# Access (same as list)
point[0]   # 10
point[1]   # 20

# Immutable!
# point[0] = 5  # TypeError!

# Use as dict keys (lists cannot be dict keys!)
locations = {(0, 0): "origin", (1, 2): "point A"}

# Named tuples (better than plain tuples)
from collections import namedtuple
Point = namedtuple('Point', ['x', 'y'])
p = Point(10, 20)
p.x   # 10 (readable!)
p.y   # 20
```

---

## 🛠️ Collections Module

```python
from collections import defaultdict, Counter, deque, OrderedDict

# --- defaultdict (auto-creates missing keys) ---
# Normal dict: person["missing_key"] → KeyError!
# defaultdict: auto-creates with default value

word_count = defaultdict(int)  # Default = 0
for word in "hello world hello python".split():
    word_count[word] += 1
# {'hello': 2, 'world': 1, 'python': 1}

# Group items
groups = defaultdict(list)
students = [("A", "Alice"), ("B", "Bob"), ("A", "Charlie")]
for grade, name in students:
    groups[grade].append(name)
# {'A': ['Alice', 'Charlie'], 'B': ['Bob']}

# --- Counter (count elements) ---
text = "hello world"
Counter(text)  # Counter({'l': 3, 'o': 2, 'h': 1, 'e': 1, ...})

nums = [1, 1, 2, 2, 2, 3]
c = Counter(nums)
c.most_common(2)  # [(2, 3), (1, 2)] — top 2 most common

# --- deque (double-ended queue) ---
# O(1) append/pop from BOTH ends (list is O(n) for left!)
dq = deque([1, 2, 3])
dq.appendleft(0)  # O(1)! — list.insert(0, x) is O(n)
dq.popleft()      # O(1)! — list.pop(0) is O(n)
dq.rotate(1)      # Rotate right: [3, 1, 2]

# Bounded deque (auto-removes oldest)
last_5 = deque(maxlen=5)
for i in range(10):
    last_5.append(i)
# deque([5, 6, 7, 8, 9]) — keeps only last 5!
```

---

## 📋 Time Complexity | நேர சிக்கல்

| Operation | List | Dict | Set |
|-----------|------|------|-----|
| Access by index | O(1) | - | - |
| Search | O(n) | O(1) | O(1) |
| Insert end | O(1) | O(1) | O(1) |
| Insert front | O(n) | - | - |
| Delete | O(n) | O(1) | O(1) |

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────┐
│      DATA STRUCTURES CHEAT SHEET                 │
├──────────────────────────────────────────────────┤
│ WHEN TO USE WHAT:                                │
│   List  = ordered, allow duplicates, by index    │
│   Dict  = key-value lookup, O(1) access          │
│   Set   = unique elements, O(1) membership       │
│   Tuple = immutable, dict keys, function returns │
│   Deque = queue/stack (O(1) both ends)          │
│                                                  │
│ INTERVIEW PATTERNS:                              │
│   "Find duplicates" → Set                        │
│   "Count frequency" → Counter / defaultdict(int) │
│   "Group by" → defaultdict(list)                 │
│   "Top K elements" → Counter.most_common(k)      │
│   "Sliding window" → deque                       │
│   "Two sum" → Dict (value → index)              │
└──────────────────────────────────────────────────┘
```

---

## 🎤 Interview Q&A | நேர்முகத் தேர்வு

**Q: List vs Tuple — when to use which?**
- List: mutable, for collections that change (shopping cart, results)
- Tuple: immutable, for fixed data (coordinates, DB rows, dict keys)
- Tuple is slightly faster and uses less memory

**Q: How does dict achieve O(1) lookup?**
- Hash table internally. Key → hash function → bucket index → value
- Collisions handled by probing. Average case O(1), worst case O(n)

**Q: When to use defaultdict vs regular dict?**
- defaultdict: when you need auto-initialization (counting, grouping)
- Avoids checking `if key in dict` before every operation

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] List vs Dict vs Set time complexity explain முடியும்
- [ ] Dict comprehension write முடியும்
- [ ] Set operations (union, intersection) use முடியும்
- [ ] Counter, defaultdict, deque use முடியும்
- [ ] Choose right data structure for a problem
