## $ Operator (Function Application)

### Definition:
```haskell
($) :: (a -> b) -> a -> b
f $ x = f x
```

### Purpose:

Replaces parentheses. Everything to the right of `$` becomes an argument to the function on the left.

### Key point:

`$` has the **lowest precedence** of all operators (evaluated last when grouping with parentheses).

### Examples
```haskell
-- Without $
sum (map sqrt [1..130])
sqrt (3 + 4 + 9)
sum (filter (> 10) (map (*2) [2..10]))

-- With $ (cleaner)
sum $ map sqrt [1..130]
sqrt $ 3 + 4 + 9
sum $ filter (> 10) $ map (*2) [2..10]
```

### Pattern:

`f $ g $ h x` is the same as `f (g (h x))`

## $ as a Function (Partial Application)

**Advanced usage:** `$` can be partially applied!
```haskell
map ($ 3) [(4+), (10*), (^2), sqrt]
-- Result: [7.0, 30.0, 9.0, 1.73...]
```

**Explanation:**

- `($ 3)` is a function with type `(a -> b) -> b`
- It takes a function and applies argument `3` to it
- `($ 3) (4+)` becomes `(4+) $ 3` which is `4 + 3 = 7`

**Important:**

- `(3)` is just a number
- `($ 3)` is a function that *"feeds 3"* to another function

## . Operator (Function Composition)

### Definition:
```haskell
(.) :: (b -> c) -> (a -> b) -> a -> c
f . g = \x -> f (g x)
```

### Purpose:

Creates a **new function** by composing two functions. `(f . g) x` means *"apply g first, then f"*.

### Key point:

`.` creates a function, not a result!

## . vs Lambda

### With lambda:
```haskell
map (\x -> negate (abs x)) [5, -3, -6]
map (\xs -> negate (sum (tail xs))) [[1..5], [3..6]]
```

### With composition (cleaner):
```haskell
map (negate . abs) [5, -3, -6]
map (negate . sum . tail) [[1..5], [3..6]]
```

Both produce the same result, but composition is more concise.

### Examples
```haskell
-- Basic composition
negate . abs           -- function that makes any number negative
negate . abs $ (-5)    -- applies to -5, result: -5

-- Multiple composition (right-associative)
f . g . h $ x          -- same as f (g (h x))

-- Practical example
oddSquareSum = sum . takeWhile (<10000) . filter odd . map (^2) $ [1..]
```

## Operator Precedence (IMPORTANT!)

### From highest to lowest:

1. **Function application (space)** - highest precedence
2. **. (composition)** - medium precedence
3. **$** - lowest precedence

### What this means:

- Precedence determines how **parentheses are grouped**, NOT execution order
- High precedence = grouped first with parentheses

### Example:
```haskell
negate . abs $ (-5)
-- Groups as: (negate . abs) $ (-5)
-- Because $ has lowest precedence, everything left groups together first
```

### Common mistake:
```haskell
negate . abs (-5)    -- ERROR! Tries to compose negate with number 5
negate . abs $ (-5)  -- CORRECT! $ separates function from argument
```

## . and $ Together

### Pattern:

Use `.` to chain functions, then `$` to apply to data
```haskell
-- Nested parentheses
sum (filter (> 10) (map (*2) [2..10]))

-- With . and $
sum . filter (> 10) . map (*2) $ [2..10]
```

### How to convert:

1. Identify the final argument (rightmost data)
2. Put `$` before it
3. Replace remaining parentheses with `.`
4. Remove the last parameter from each function

### Example conversion:
```haskell
replicate 100 (product (map (*3) (zipWith max [1,2] [4,5])))

-- becomes
replicate 100 . product . map (*3) . zipWith max [1,2] $ [4,5]
```

## When to Use What

**Use $:**

- To avoid parentheses in simple expressions
- When applying a function to a complex argument
- `f $ big expression` instead of `f (big expression)`

**Use .:**

- To create reusable function pipelines
- In point-free style (no explicit parameters)
- When chaining multiple transformations

**Use both:**

- Complex pipelines: chain functions with `.`, apply to data with `$`
- Pattern: `f . g . h $ data`

**Don't overuse:**

- Very long chains reduce readability
- Consider `let` bindings for complex logic
- Balance between conciseness and clarity