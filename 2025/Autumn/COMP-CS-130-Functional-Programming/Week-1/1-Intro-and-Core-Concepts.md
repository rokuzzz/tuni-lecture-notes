## What is Haskell?

**Haskell is a purely functional programming language** - fundamentally different from imperative languages like C++, Java, Python. The core difference is philosophical: instead of telling the computer what to do step by step, you tell it what things are.

## Core Characteristics

### 1. Purely Functional

Functions have no side effects and can only calculate and return results. If you say `a = 5`, then `a` is always 5. Same input always gives same output (referential transparency).

### 2. Lazy Evaluation

Haskell doesn't calculate anything until it has to show you a result. When you write `doubleMe(doubleMe(doubleMe(xs)))`, it only does the work when you actually need the answer, not three separate passes.

### 3. Static Typing with Inference

The compiler catches type errors early, but you don't need to write types everywhere. Haskell figures out that `a = 5 + 4` means `a` is a number automatically.

## Development Workflow

### GHCi (Interactive Mode)
```bash
ghci                   # Start interactive mode
:l filename            # Load functions from filename.hs
:r                     # Reload current file
:q                     # Quit
```

### Typical Workflow:

1. Write functions in .hs file
2. Load in GHCi with `:l filename`
3. Test functions interactively
4. Modify .hs file
5. Reload with `:r` or `:l filename`
6. Repeat

## Mental Shift: What vs How

The biggest change is shifting from "how to do something" to "what something is". Instead of thinking "factorial is a loop that multiplies numbers from 1 to n", think "factorial is the product of all numbers from 1 to that number". You're defining relationships and transformations rather than giving step-by-step instructions.

### From "How to do it" → "What it is"

- Factorial isn't "multiply numbers from 1 to n in a loop"
- Factorial *is* "product of all numbers from 1 to that number"
- Express relationships and definitions, not step-by-step instructions

## Remember

- Functions have **no side effects**
- Variables are **immutable**
- Evaluation is **lazy**
- Types are **inferred automatically**
- Think in terms of **transformations on data**