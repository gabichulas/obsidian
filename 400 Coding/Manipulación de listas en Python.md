# Python List Manipulation Cheatsheet

## Basic Slicing

Syntax: `lista[inicio:fin:paso]`

```python
nums = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
nums[2:5]    # [2, 3, 4] (from index 2 to 4)
nums[:5]     # [0, 1, 2, 3, 4] (start to index 4)
nums[5:]     # [5, 6, 7, 8, 9] (index 5 to end)
nums[::2]    # [0, 2, 4, 6, 8] (every 2nd item)
nums[::-1]   # [9, 8, 7, 6, 5, 4, 3, 2, 1, 0] (reverse)
nums[-3:]    # [7, 8, 9] (last 3 items)
```

---
## Advanced Slicing

```python
# Step slicing
nums[1::2]   # [1, 3, 5, 7, 9] (odd indexes)
nums[5:2:-1] # [5, 4, 3] (reverse slice)

# Modifications
nums[2:5] = [10, 11, 12]  # Replace sublist
nums[3:3] = [20, 21, 22]  # Insert without replacing
nums[2:5] = []            # Delete elements

# Copying
shallow_copy = nums[:]    # Shallow copy
```

---
## List Operations

```python
# Rotation
rot_left = nums[1:] + nums[:1]   # [1, 2, 3, 4, 0]
rot_right = nums[-1:] + nums[:-1] # [4, 0, 1, 2, 3]

# Partitioning
half = len(nums)//2
first_half, second_half = nums[:half], nums[half:]

# Matrix operations
matrix = [[1,2,3],[4,5,6],[7,8,9]]
first_col = [row[0] for row in matrix]  # [1, 4, 7]

# Alternating elements
evens = nums[::2]    # [0, 2, 4, 6, 8]
odds = nums[1::2]     # [1, 3, 5, 7, 9]
```

---
## Special Cases

```python
# Empty slices
nums[10:]      # [] (no IndexError)
nums[:100]     # full list (no IndexError)

# Negative indexes
nums[-1]       # 9 (last element)
nums[-4:-1]    # [6, 7, 8]

# Shallow copy caveat
nested = [[1,2], [3,4]]
copy = nested[:]
copy[0][0] = 99  # Affects both!
```

