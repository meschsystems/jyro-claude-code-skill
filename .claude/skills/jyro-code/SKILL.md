---
name: jyro-code
description: Use when writing, debugging, editing, or optimizing Jyro scripts. Apply whenever the user is working with .jyro files, asks about Jyro syntax, or mentions Jyro scripting. Also use when the active file has a .jyro extension.
---

# Jyro Code Expert Skill

You are an expert Jyro programmer. Your role is to help users write perfect, idiomatic Jyro scripts that solve their business problems efficiently and correctly.

## Language Overview

Jyro is an imperative, optionally-typed scripting language designed for secure, sandboxed data transformation. Key characteristics:

- **Data-centric**: All input/output happens through the implicit `Data` context object
- **JSON-native**: Works directly with JSON-compatible types (number, string, boolean, object, array, null)
- **Sandboxed**: No file I/O, network access, or system calls
- **Resource-limited**: Enforced limits on execution time, statements, loops, and stack depth
- **Immutable stdlib**: All standard library functions return new values — they never mutate their inputs

---

## 1. Program Structure and the Data Context

A Jyro script is a flat sequence of statements executed top-to-bottom. There are no modules, imports, or user-defined named functions.

**The `Data` object** is the sole input/output mechanism:

```jyro
# Data is always available — never declare it
var input = Data.userInput       # Read from Data (provided by host)
Data.processed = true            # Write to Data (returned to host)
Data.timestamp = Now()

# Local variables are discarded after execution
var temp = 100                   # NOT returned to host
Data.output = temp               # This IS returned
```

**Rules:**
- `Data` is always an object — never null, never reassignable
- Only modifications to `Data` persist after execution
- Local variables (`var`) exist only during script execution
- `Data` is always returned to the host, even on failure

**Creating nested structure — intermediate objects are required:**
```jyro
# WRONG — error if Data.level1 doesn't exist
Data.level1.level2.value = "x"

# RIGHT — create each level explicitly
Data.level1 = {}
Data.level1.level2 = {}
Data.level1.level2.value = "x"

# Or use a literal
Data.level1 = {"level2": {"value": "x"}}
```

---

## 2. Comments

Line comments start with `#`. There are no block comments.

```jyro
# This is a comment
var x = 10  # Inline comment
```

---

## 3. Literals

| Type | Examples |
|------|---------|
| Number (int) | `42`, `-5`, `0` |
| Number (float) | `3.14`, `-0.5` |
| Number (hex) | `0xFF`, `0xff` |
| Number (binary) | `0b1010`, `0b11111111` |
| String | `"hello"`, `'world'` |
| Boolean | `true`, `false` |
| Null | `null` |
| Array | `[1, 2, 3]`, `[]` |
| Object | `{name: "Alice", age: 30}`, `{}` |

### String escape sequences

`\n` `\r` `\t` `\\` `\"` `\'` `\0` `\uXXXX`

Both single and double quotes are supported and behave identically. There is **no string interpolation** — use concatenation:

```jyro
var greeting = "Hello, " + name   # Correct
# var greeting = "Hello, ${name}" # WRONG — no interpolation
```

### Object literal keys

Object keys can be unquoted identifiers or quoted strings:

```jyro
var obj = {name: "Alice", age: 30}           # Unquoted keys
var obj2 = {"first-name": "Alice"}           # Quoted keys (for special chars)
```

---

## 4. Variables and Type Hints

```jyro
var count = 0                     # Initialized
var name = "Alice"
var unset                         # Defaults to null
var items: array = [1, 2, 3]     # With type hint
var label: string = 42            # Coerced to "42"
```

### Type hints

Optional type annotations: `number`, `string`, `boolean`, `array`, `object`. Typed variables **must** have initial values. When present, values are coerced on assignment:

- To `number`: strings are parsed, booleans become `1`/`0`
- To `string`: numbers and booleans become their string form
- To `boolean`: numbers (`0` → `false`, else `true`), strings (`"true"`/`"false"` only, case-insensitive)
- To `array`, `object`: no cross-type coercion — must match exactly

Type hints are enforced on every assignment, and throw runtime errors when the value does not match the type hint and cannot be coerced.

### Variable scoping

- Variables declared before a block are visible inside the block
- Modifications inside blocks affect the outer scope
- Loop variables (`for`, `foreach`) are scoped to the loop body
- Inner `var` declarations can shadow outer variables

---

## 5. Operators

### Precedence (highest to lowest)

| # | Operators | Description | Associativity |
|---|-----------|-------------|---------------|
| 1 | `()` `.` `[]` `x++` `x--` | Call, access, postfix inc/dec | Left |
| 2 | `++x` `--x` `-x` | Prefix inc/dec, negate | Right |
| 3 | `*` `/` `%` | Multiplicative | Left |
| 4 | `+` `-` | Additive | Left |
| 5 | `<` `<=` `>` `>=` | Relational | Left |
| 6 | `==` `!=` | Equality | Left |
| 7 | `is` `is not` | Type checking | — |
| 8 | `not` | Logical NOT | Right |
| 9 | `and` | Logical AND (short-circuit) | Left |
| 10 | `or` | Logical OR (short-circuit) | Left |
| 11 | `??` | Null coalescing | Right |
| 12 | `? :` | Ternary conditional | Right |

### Assignment operators

`=`, `+=`, `-=`, `*=`, `/=`, `%=`

Assignment targets: identifiers, property access (`obj.prop`), index access (`arr[0]`, `obj["key"]`).

### Increment/decrement

```jyro
x++     # Postfix: returns old value, then increments
x--     # Postfix: returns old value, then decrements
++x     # Prefix: increments, then returns new value
--x     # Prefix: decrements, then returns new value
```

### Arithmetic

- `+` on two numbers → addition; if either operand is a string → string concatenation (auto-converts numbers, booleans, and null but **not** arrays or objects)
- `-`, `*`, `/`, `%` → numeric only
- **Division/modulo by zero throws a runtime error** — always validate

```jyro
var sum = 10 + 5              # 15
var msg = "Count: " + 42     # "Count: 42" (auto-converts to string)
if divisor != 0 then
    Data.result = value / divisor
end
```

### Comparison

- Numbers compare numerically, strings compare lexicographically (ordinal, case-sensitive)
- Relational operators (`<`, `<=`, `>`, `>=`) require both operands to be the same comparable type — comparing across types (e.g. `42 < "100"`) throws a runtime error
- `==`/`!=` use deep value equality for all types, including arrays and objects; cross-type equality always returns `false` (no coercion)
- `null` is never equal to anything, including itself — use `is null` to test for null

### Null coalescing

```jyro
var name = Data.user.name ?? "Unknown"    # "Unknown" if name is null
var first = null ?? null ?? "found"       # "found"
```

### Ternary conditional

```jyro
var status = Data.active ? "on" : "off"
var grade = score >= 90 ? "A" : score >= 80 ? "B" : "C"   # Nestable
```

---

## 6. Truthiness

| Value | Truthy? |
|-------|---------|
| `null` | **false** |
| `false` | **false** |
| `0` / `0.0` | **false** |
| `""` (empty string) | **false** |
| `true`, non-zero numbers, non-empty strings | **true** |
| Arrays — even empty `[]` | **true** |
| Objects — even empty `{}` | **true** |

**CRITICAL**: Empty arrays and objects are truthy!

```jyro
# WRONG — executes even if items is empty []
if Data.items then ... end

# RIGHT — check length explicitly
if Length(Data.items) > 0 then ... end
```

### Logical operator return values

`and` and `or` return one of their operand values, not necessarily a boolean. `not` always returns a boolean.

```jyro
"hello" and "world"          # "world" (both truthy → returns last)
null and "skipped"            # null (first falsy → returns it)
null or "fallback"            # "fallback" (first falsy → returns second)
"first" or "second"           # "first" (first truthy → returns it)
not null                      # true
not "hello"                   # false
```

---

## 7. Type Checking

```jyro
value is number               # true if value is a number
value is not string            # negated type check
```

Valid type keywords: `number`, `string`, `boolean`, `array`, `object`, `null`.

```jyro
if Data.input is number then
    Data.result = Data.input * 2
elseif Data.input is string then
    Data.result = ToUpper(Data.input)
end
```

---

## 8. Control Flow

### if / elseif / else

```jyro
if condition then
    # body
elseif otherCondition then
    # body
else
    # body
end
```

The `then` keyword is **required** after each condition. The block is terminated by `end`.

### `elseif` vs `else if`

These are both valid but semantically different:

**`elseif` (one word)** — part of a single if statement, one `end`:
```jyro
if score >= 90 then
    grade = "A"
elseif score >= 80 then
    grade = "B"
else
    grade = "F"
end
```

**`else if` (two words)** — nested if inside else, needs TWO `end` keywords:
```jyro
if score >= 90 then
    grade = "A"
else
    if score >= 80 then
        grade = "B"
    else
        grade = "F"
    end       # closes inner if
end           # closes outer if
```

Use `elseif` (recommended) for simple conditional chains. Use `else if` only when you need to declare variables in the else block before the nested condition.

### while

```jyro
while condition do
    # body
end
```

### for (range-based)

```jyro
for i in 0 to 5 do              # 0, 1, 2, 3, 4 (exclusive upper bound)
    # body
end

for i in 0 to 10 by 2 do        # 0, 2, 4, 6, 8 (with step)
    # body
end

for i in 5 downto 0 do          # 5, 4, 3, 2, 1 (exclusive lower bound)
    # body
end

for i in 10 downto 0 by 2 do    # 10, 8, 6, 4, 2 (with step)
    # body
end
```

**Bounds are always exclusive** — like `<` for `to`, `>` for `downto`. The step must always be a positive number; `by -2` or `by 0` causes a runtime error. For `downto`, the step is subtracted automatically. For inclusive bounds, adjust by 1:

```jyro
for i in 1 to 10 + 1 do         # 1 through 10 inclusive
    # body
end

for i in 10 downto 1 - 1 do     # 10 through 1 inclusive
    # body
end
```

The loop variable is scoped to the loop body. Start, end, and step expressions are evaluated once at loop entry.

### foreach

```jyro
foreach item in Data.items do    # Iterates over array elements
    item.processed = true
end

foreach ch in "hello" do         # Iterates over characters (as strings)
    # ch is "h", "e", "l", "l", "o"
end

foreach val in Data.scores do    # Iterates over object VALUES
    total += val
end
```

To iterate over object keys: `foreach key in Keys(obj) do ... end`

Only arrays, strings, and objects are iterable. Attempting to iterate over a number, boolean, or null causes a runtime error.

### switch

```jyro
switch Data.action do
    case "create" then
        Data.status = "created"
    case "update", "patch" then
        Data.status = "updated"
    default then
        Data.status = "unknown"
end
```

Each `case` block is implicitly terminated by the next `case`, `default`, or the closing `end` of the `switch` statement. Multiple values per case are comma-separated. `default` is optional — if no case matches and there is no `default`, execution silently continues past the `switch`. There is no fall-through — only the matched case executes.

### Loop control

- `break` — exits the current (innermost) loop
- `continue` — skips to the next iteration

### return and fail

Both terminate the script immediately. They accept an optional **string message** on the **same line**.

```jyro
return                              # Clean exit, no message
return "Processing complete"        # Clean exit with message
fail                                # Failure exit, default message
fail "Invalid input"                # Failure exit with message
```

| Keyword | Script Succeeds? | Message Severity |
|---------|-----------------|------------------|
| `return` | Yes | Info (if message) |
| `fail` | No | Error |

**CRITICAL**: The message must be on the **same line** as the keyword:
```jyro
# CORRECT
return "Done"

# WRONG — "Done" is a separate statement, not a message
return
"Done"
```

`Data` is **always** returned to the host with all mutations, regardless of whether the script returns or fails.

---

## 9. Lambda Expressions

Lambdas are anonymous functions used as arguments to higher-order stdlib functions.

```jyro
x => x * 2                          # Single parameter
(x, y) => x + y                     # Multiple parameters
(acc, item) => acc + item.price      # Used with Reduce
```

The body is a **single expression** (not a block of statements). Closures capture outer variables.

```jyro
var threshold = 100
var big = Where(Data.items, x => x > threshold)    # Captures threshold

var doubled = Map(Data.numbers, x => x * 2)
var evens = Where(Data.numbers, x => x % 2 == 0)
var total = Reduce(Data.numbers, (acc, x) => acc + x, 0)
```

---

## 10. Property and Index Access

```jyro
obj.name                 # Property access (dot notation)
obj.address.city         # Nested property traversal
arr[0]                   # Array index (0-based)
obj["key"]               # Object bracket access (for dynamic keys)
str[0]                   # String character access (returns single-char string)
```

- Property access returns `null` if the property doesn't exist
- Index access returns `null` if out of bounds or key is missing
- Accessing a property on `null` returns `null` (null propagation) — so `Data.user.profile.name` returns `null` if any part of the chain is null
- Assigning a property on `null` causes a runtime error
- Properties and indexes are assignable: `obj.name = "Bob"`, `arr[0] = 99`
- Keywords are valid as property names: `Data.for = "x"`, `Data.number = 42`

---

## 11. Standard Library — String Functions (18)

All function names are **PascalCase** and **case-sensitive**.

| Function | Parameters | Returns | Description |
|----------|-----------|---------|-------------|
| `ToUpper(s)` | string | string | Uppercase (invariant culture) |
| `ToLower(s)` | string | string | Lowercase (invariant culture) |
| `Trim(s)` | string | string | Strip leading/trailing whitespace |
| `Replace(s, old, new)` | string, string, string | string | Replace all occurrences |
| `Contains(haystack, needle)` | any, any | boolean | Substring check (strings) or element check (arrays) |
| `StartsWith(s, prefix)` | string, string | boolean | Prefix check |
| `EndsWith(s, suffix)` | string, string | boolean | Suffix check |
| `Split(s, delim)` | string, string | array | Split into array of strings |
| `Join(arr, sep)` | array, string | string | Join elements into string |
| `Substring(s, start, len?)` | string, number[, number] | string | Extract substring; len optional (to end) |
| `ToNumber(s)` | string | number or **null** | Parse number; **returns null if invalid** |
| `PadLeft(s, width, char?)` | string, number[, string] | string | Left-pad; char defaults to space |
| `PadRight(s, width, char?)` | string, number[, string] | string | Right-pad; char defaults to space |
| `RegexTest(s, pattern)` | string, string | boolean | Test regex match |
| `RegexMatch(s, pattern)` | string, string | string or null | First match or null |
| `RegexMatchAll(s, pattern)` | string, string | array | All matches as array |
| `RegexMatchDetail(s, pattern)` | string, string | object or null | `{match, index, groups}` or null |
| `RandomString(len, charset?)` | number[, string] | string | Random string; charset defaults to alphanumeric |

**Regex pattern escaping**: Jyro's string lexer only recognises the escape sequences listed above (`\n`, `\r`, `\t`, `\\`, `\"`, `\'`, `\0`, `\uXXXX`). Any other backslash sequence — including `\d`, `\w`, and `\s` — causes a **parse error**. Use character classes instead:

| Instead of | Use |
|------------|-----|
| `\d` | `[0-9]` |
| `\w` | `[a-zA-Z0-9_]` |
| `\s` | `[ \t\n\r]` |
| `\.` | `[.]` |

```jyro
# WRONG — parse error: \d is not a recognised escape sequence
var num = RegexMatch(text, "\d+")

# RIGHT — use character class
var num = RegexMatch(text, "[0-9]+")
```

---

## 12. Standard Library — Array Functions (23)

**All array functions return new arrays — they never mutate the input.**

| Function | Parameters | Returns | Description |
|----------|-----------|---------|-------------|
| `Length(v)` | any | number | String length, array length, object key count; 0 for null |
| `Append(arr, val)` | array, any | array | New array with value appended |
| `Prepend(arr, val)` | array, any | array | New array with value prepended |
| `First(arr)` | array | any | First element or null if empty |
| `Last(arr)` | array | any | Last element or null if empty |
| `IndexOf(source, search)` | any, any | number | Index in array or position in string; -1 if not found |
| `Reverse(arr)` | array | array | New reversed array |
| `Slice(arr, start, end?)` | array, number[, number] | array | Extract elements; end defaults to length |
| `Concatenate(a, b)` | array, array | array | Join two arrays |
| `Distinct(arr)` | array | array | Remove duplicates (first occurrence kept) |
| `Sort(arr)` | array | array | Sort: nulls first, then numbers, strings, booleans |
| `Flatten(arr)` | array | array | Recursively flatten nested arrays |
| `FlattenOnce(arr)` | array | array | Flatten one level only |
| `Range(start, end, step?)` | number, number[, number] | array | Generate number sequence (inclusive) |
| `Skip(arr, n)` | array, number | array | Skip first n elements |
| `Insert(arr, idx, val)` | array, number, any | array | New array with value at index |
| `RemoveAt(arr, idx)` | array, number | array | New array without element at index |
| `RemoveFirst(arr)` | array | array | New array without first element |
| `RemoveLast(arr)` | array | array | New array without last element |
| `RandomChoice(arr)` | array | any | Random element or null if empty |
| `SortByField(arr, field, dir)` | array, string, string | array | Sort objects by field; dir: `"asc"` or `"desc"` |
| `GroupBy(arr, field)` | array, string | object | Group objects by field value into `{key: [items]}` |
| `SelectMany(arr, field)` | array, string | array | Flatten array fields from objects |

**Key behaviors:**
- `Sort` order for mixed types: null → numbers → strings → booleans
- `GroupBy` uses the field's string representation as the group key; null values grouped under `"null"`
- Array concatenation also works with `+` operator: `[1, 2] + [3, 4]` → `[1, 2, 3, 4]`

---

## 13. Standard Library — Math Functions (14)

| Function | Parameters | Returns | Description |
|----------|-----------|---------|-------------|
| `Abs(n)` | number | number | Absolute value |
| `Floor(n)` | number | number | Round down |
| `Ceiling(n)` | number | number | Round up |
| `Min(arr)` | array | number or null | Minimum number in array; null if no numbers |
| `Max(arr)` | array | number or null | Maximum number in array; null if no numbers |
| `Power(base, exp)` | number, number | number | Exponentiation |
| `SquareRoot(n)` | number | number | Square root |
| `Log(n, base?)` | number[, number] | number | Natural log if no base; log with specified base |
| `Sum(arr)` | array | number | Sum of numbers; non-numbers ignored; 0 if none |
| `Average(arr)` | array | number or null | Mean of numbers; **null if empty or no numbers** |
| `Clamp(n, min, max)` | number, number, number | number | Constrain to range |
| `Median(arr)` | array | number or null | Median value; null if no numbers |
| `Mode(arr)` | array | number or null | Most frequent number; null if no numbers |
| `RandomInt(min, max)` | number, number | number | Random integer (inclusive range) |

**CRITICAL**: `Min`, `Max`, `Sum`, `Average`, `Median`, and `Mode` all take a **single array argument** — not multiple arguments:

```jyro
# WRONG
var total = Sum(1, 2, 3)
var minimum = Min(a, b)

# RIGHT
var total = Sum([1, 2, 3])
var minimum = Min([a, b])
```

---

## 14. Standard Library — Date/Time Functions (7)

All dates are strings in ISO 8601 format (`yyyy-MM-ddTHH:mm:ss.fffZ`).

| Function | Parameters | Returns | Description |
|----------|-----------|---------|-------------|
| `Now()` | (none) | string | Current UTC datetime |
| `Today()` | (none) | string | Current UTC date (`yyyy-MM-dd`) |
| `ParseDate(s)` | string | string | Parse various formats to ISO 8601 |
| `FormatDate(date, fmt)` | string, string | string | Format with custom pattern |
| `DateAdd(date, amount, unit)` | string, number, string | string | Add time to date |
| `DateDiff(end, start, unit)` | string, string, string | number | Difference between dates |
| `DatePart(date, part)` | string, string | number | Extract component |

**DateAdd units**: `"days"`, `"weeks"`, `"months"`, `"years"`, `"hours"`, `"minutes"`, `"seconds"` (singular forms also accepted)

**DatePart parts**: `"year"`, `"month"`, `"day"`, `"hour"`, `"minute"`, `"second"`, `"dayofweek"` (0=Sunday), `"dayofyear"`

**Format specifiers**: `yyyy` (year), `MM` (month), `dd` (day), `HH` (24-hour), `mm` (minute), `ss` (second)

```jyro
var tomorrow = DateAdd(Today(), 1, "days")
var daysBetween = DateDiff(endDate, startDate, "days")
var year = DatePart(Now(), "year")
var formatted = FormatDate(Now(), "yyyy-MM-dd HH:mm")
```

---

## 15. Standard Library — Query Functions (8)

Field-based operations on arrays of objects. No lambdas — use string field names and operator strings.

Supported operators: `"=="`, `"!="`, `"<"`, `"<="`, `">"`, `">="`.

| Function | Parameters | Returns | Description |
|----------|-----------|---------|-------------|
| `WhereByField(arr, field, op, val)` | array, string, string, any | array | Filter objects by field condition |
| `FindByField(arr, field, op, val)` | array, string, string, any | any | First match or null |
| `AnyByField(arr, field, op, val)` | array, string, string, any | boolean | Any object matches? |
| `AllByField(arr, field, op, val)` | array, string, string, any | boolean | All objects match? |
| `CountIf(arr, field, op, val)` | array, string, string, any | number | Count matching objects |
| `Select(arr, field)` | array, string | array | Extract one field from all objects |
| `Project(arr, fields)` | array, array | array | Keep only listed fields |
| `Omit(arr, fields)` | array, array | array | Remove listed fields |

**Dot-notation** is supported for nested fields: `WhereByField(users, "address.city", "==", "Boston")`

```jyro
# Filter, find, check
var active = WhereByField(Data.users, "status", "==", "active")
var admin = FindByField(Data.users, "role", "==", "admin")
var hasAdmin = AnyByField(Data.users, "role", "==", "admin")
var allVerified = AllByField(Data.users, "verified", "==", true)
var activeCount = CountIf(Data.users, "status", "==", "active")

# Field extraction
var names = Select(Data.users, "name")
var summary = Project(Data.users, ["id", "name", "email"])
var sanitized = Omit(Data.users, ["password", "token"])
```

---

## 16. Standard Library — Lambda / Higher-Order Functions (8)

These take lambda expressions as callbacks.

| Function | Parameters | Returns | Description |
|----------|-----------|---------|-------------|
| `Map(arr, fn)` | array, function | array | Transform each element |
| `Where(arr, fn)` | array, function | array | Filter by predicate |
| `Reduce(arr, fn, init)` | array, function, any | any | Fold; callback is `(acc, item) => result` |
| `Each(arr, fn)` | array, function | null | Side-effect iteration |
| `Find(arr, fn)` | array, function | any | First match or null |
| `All(arr, fn)` | array, function | boolean | All elements satisfy predicate? |
| `Any(arr, fn)` | array, function | boolean | Any element satisfies predicate? |
| `SortBy(arr, fn)` | array, function | array | Sort by computed key |

```jyro
# Transform
var doubled = Map(Data.numbers, x => x * 2)
var upper = Map(Data.names, n => ToUpper(n))

# Filter
var adults = Where(Data.users, u => u.age >= 18)
var evens = Where(Data.numbers, x => x % 2 == 0)

# Aggregate
var total = Reduce(Data.items, (sum, item) => sum + item.price, 0)

# Search
var first = Find(Data.items, x => x.status == "pending")

# Check
var anyExpired = Any(Data.items, x => x.expired)
var allValid = All(Data.items, x => x.price > 0)

# Sort
var byPrice = SortBy(Data.items, x => x.price)
```

### Lambda vs Query: When to use which

- **Query functions** (`WhereByField`, `FindByField`, etc.): Best for simple single-field comparisons — concise and readable
- **Lambda functions** (`Where`, `Find`, etc.): Best for complex conditions, computed values, or multi-field logic

```jyro
# Simple: prefer query
var active = WhereByField(Data.users, "active", "==", true)

# Complex: prefer lambda
var eligible = Where(Data.users, u => u.age >= 18 and u.active and u.balance > 0)
```

---

## 17. Standard Library — Utility Functions (14)

| Function | Parameters | Returns | Description |
|----------|-----------|---------|-------------|
| `TypeOf(v)` | any | string | `"null"`, `"boolean"`, `"number"`, `"string"`, `"array"`, `"object"` |
| `Coalesce(arr)` | array | any | First non-null element, or null |
| `ToString(v)` | any | string | Convert to string |
| `ToBoolean(v)` | any | boolean | Convert using truthiness rules |
| `Keys(obj)` | object | array | Property names as array |
| `Values(obj)` | object | array | Property values as array |
| `HasProperty(obj, key)` | object, string | boolean | Property exists? |
| `Merge(arr)` | array of objects | object | Merge objects; later overrides earlier |
| `Clone(v)` | any | any | Deep clone |
| `NewGuid()` | (none) | string | Lowercase hyphenated GUID |
| `Base64Encode(s)` | string | string | Encode to Base64 |
| `Base64Decode(s)` | string | string | Decode from Base64 |
| `FromJson(s)` | string | any | Parse JSON; null on error |
| `ToJson(v)` | any | string | Serialize to JSON |

**CRITICAL — `Merge` takes an array of objects**, not variadic arguments:

```jyro
# WRONG
var combined = Merge(defaults, overrides)

# RIGHT
var combined = Merge([defaults, overrides])

# Multiple objects
var result = Merge([base, layer1, layer2])   # Later keys override earlier

# Shallow copy
var copy = Merge([original])
```

**`Merge` is shallow** — nested objects are replaced, not deep-merged:
```jyro
var a = {nested: {x: 1, y: 2}}
var b = {nested: {z: 3}}
var result = Merge([a, b])
# result.nested = {z: 3}  — NOT {x: 1, y: 2, z: 3}
```

**`Coalesce` also takes an array:**
```jyro
var value = Coalesce([Data.primary, Data.fallback, "default"])
```

---

## 18. Standard Library — Schema Validation (2)

| Function | Parameters | Returns | Description |
|----------|-----------|---------|-------------|
| `ValidateRequired(obj, fields)` | object, array | object | Returns `{valid: boolean, errors: array}` |
| `ValidateSchema(data, schema)` | any, object | array | Validate against JSON Schema; returns error array |

```jyro
# Check required fields
var result = ValidateRequired(Data, ["name", "email", "age"])
if not result.valid then
    fail Join(result.errors, "; ")
end

# JSON Schema validation
var errors = ValidateSchema(Data.payload, {
    "type": "object",
    "required": ["id", "name"],
    "properties": {
        "id": {"type": "number"},
        "name": {"type": "string"}
    }
})
if Length(errors) > 0 then
    fail "Schema errors: " + ToJson(errors)
end
```

---

## 19. Edge Cases Quick Reference

### Return values on empty/missing data

| Function | Condition | Returns |
|----------|-----------|---------|
| `First(arr)` | empty array | `null` |
| `Last(arr)` | empty array | `null` |
| `IndexOf(arr, val)` | not found | `-1` |
| `WhereByField(...)` | no matches | `[]` (empty array) |
| `FindByField(...)` | no match | `null` |
| `Find(arr, fn)` | no match | `null` |
| `AnyByField(...)` | empty array | `false` |
| `AllByField(...)` | empty array | `true` (vacuous truth) |
| `Any(arr, fn)` | empty array | `false` |
| `All(arr, fn)` | empty array | `true` (vacuous truth) |
| `Select(...)` | empty array | `[]` |
| `SelectMany(...)` | empty array | `[]` |
| `Project(...)` | empty array | `[]` |
| `Omit(...)` | empty array | `[]` |
| `Merge(arr)` | empty array | `{}` |
| `Min(arr)` | no numbers | `null` |
| `Max(arr)` | no numbers | `null` |
| `Average(arr)` | no numbers | `null` |
| `Median(arr)` | no numbers | `null` |
| `Mode(arr)` | no numbers | `null` |
| `Sum(arr)` | no numbers | `0` |
| `ToNumber(s)` | invalid string | `null` |
| `RegexMatch(...)` | no match | `null` |
| `RegexMatchAll(...)` | no matches | `[]` |
| `RegexMatchDetail(...)` | no match | `null` |
| `RandomChoice(arr)` | empty array | `null` |

### Case sensitivity

- String comparisons (`==`, `!=`, `<`, `>`) are case-sensitive
- `Contains`, `StartsWith`, `EndsWith` are case-sensitive
- Query function comparisons are case-sensitive
- Use `ToLower()` or `ToUpper()` for case-insensitive matching

### Sort order for mixed types

`Sort` groups by type: null → numbers (ascending) → strings (lexicographic) → booleans (`false` < `true`)

### Split preserves empty strings

```jyro
Split("a,,b", ",")        # ["a", "", "b"]
Split(",a,b,", ",")       # ["", "a", "b", ""]
```

---

## 20. Idiomatic Patterns

### Input validation with guard clauses

```jyro
if Data.items is null or Data.items is not array then
    fail "Items must be an array"
end

if Length(Data.items) == 0 then
    Data.result = []
    return "No items to process"
end

if Data.email is null or Data.email == "" then
    fail "Email is required"
end

# Proceed with valid data
```

### Safe property access

```jyro
# Short-circuit prevents null errors
if Data.user is not null and Data.user.profile is not null then
    Data.userName = Data.user.profile.name
else
    Data.userName = "Guest"
end

# Null coalescing for simple defaults
var name = Data.user.name ?? "Unknown"
```

### Array processing pipeline

```jyro
# Filter → Sort → Extract
var active = WhereByField(Data.employees, "active", "==", true)
var engineering = WhereByField(active, "department", "==", "Engineering")
var sorted = SortByField(engineering, "salary", "desc")
var names = Select(sorted, "name")
Data.topEngineers = names
```

### Aggregation with Select

```jyro
# Extract field values, then aggregate
var prices = Select(Data.items, "price")
Data.total = Sum(prices)
Data.average = Average(prices)
Data.highest = Max(prices)
Data.lowest = Min(prices)
```

### Configuration defaults with Merge

```jyro
Data.config = Merge([
    {"timeout": 30, "retries": 3, "debug": false},   # Defaults
    Data.config                                        # User overrides
])
```

### Data enrichment

```jyro
Data.count = Length(Data.items)
Data.total = Sum(Select(Data.items, "price"))
Data.hasItems = Length(Data.items) > 0
Data.uniqueCategories = Distinct(Select(Data.items, "category"))
return
```

### Chained transforms

```jyro
var names = Select(Data.users, "name")
var upper = Map(names, n => ToUpper(n))
var sorted = Sort(upper)
Data.result = Join(sorted, ", ")
```

### Accumulation (when lambdas/queries aren't sufficient)

```jyro
# Manual loop for complex per-item logic
var total = 0
foreach item in Data.items do
    if item.price is number and item.taxable then
        total += item.price * 1.1
    elseif item.price is number then
        total += item.price
    end
end
Data.totalWithTax = total
```

### Collecting all nested arrays

```jyro
var allTags = SelectMany(Data.documents, "tags")
Data.uniqueTags = Distinct(allTags)
```

### API response shaping

```jyro
var activeUsers = WhereByField(Data.users, "active", "==", true)
Data.response = Project(activeUsers, ["id", "name", "email"])
Data.sanitized = Omit(Data.users, ["password", "token", "secret"])
```

---

## 21. Critical Gotchas

### 1. Empty arrays and objects are truthy
```jyro
# WRONG
if Data.items then ... end           # Always true, even for []

# RIGHT
if Length(Data.items) > 0 then ... end
```

### 2. ToNumber returns null (not 0) for invalid input
```jyro
var num = ToNumber(Data.input)
if num is null then
    fail "Invalid number: " + Data.input
end
```

### 3. Division by zero crashes the script
```jyro
if count != 0 then
    Data.average = total / count
else
    Data.average = 0
end
```

### 4. All array functions return new arrays
```jyro
var items = [3, 1, 2]
Sort(items)                # items is STILL [3, 1, 2]
var sorted = Sort(items)   # sorted is [1, 2, 3]

# Same for Append:
var items = [1, 2]
Append(items, 3)           # items is STILL [1, 2]
items = Append(items, 3)   # NOW items is [1, 2, 3]
```

### 5. Cannot modify array during iteration
```jyro
# WRONG — runtime error
foreach item in Data.items do
    Data.items = Append(Data.items, item)    # Modifying iterated collection!
end

# RIGHT — use a separate collection
var additions = []
foreach item in Data.items do
    if item.shouldDuplicate then
        additions = Append(additions, item)
    end
end
foreach addition in additions do
    Data.items = Append(Data.items, addition)
end
```

### 6. Nested properties need intermediate objects
```jyro
# WRONG
Data.deep.nested.value = "x"

# RIGHT
Data.deep = {}
Data.deep.nested = {}
Data.deep.nested.value = "x"
```

### 7. Missing `then` after `if`/`elseif`
```jyro
# WRONG: if x > 0
# RIGHT:
if x > 0 then
```

### 8. Missing `do` after `while`/`for`/`foreach`
```jyro
# WRONG: while x > 0
# RIGHT:
while x > 0 do
```

### 9. Missing `end` for blocks
Every `if`, `while`, `for`, `foreach`, and `switch` must be closed with `end`. Inside a `switch`, each `case` and `default` block is implicitly terminated by the next `case`, `default`, or the closing `end`.

### 10. No string interpolation
```jyro
# WRONG: "Hello ${name}" or "Hello {name}"
# RIGHT:
"Hello " + name
```

### 11. Min/Max/Sum/Average take arrays, not multiple arguments
```jyro
# WRONG: Sum(1, 2, 3) or Min(a, b)
# RIGHT:
Sum([1, 2, 3])
Min([a, b])
```

### 12. Merge takes an array of objects
```jyro
# WRONG: Merge(obj1, obj2)
# RIGHT:
Merge([obj1, obj2])
```

### 13. GroupBy takes a field name string, not a lambda
```jyro
# WRONG: GroupBy(arr, x => x.category)
# RIGHT:
GroupBy(arr, "category")
```

### 14. Regex escaping — `\d`, `\w`, `\s` cause parse errors
```jyro
# WRONG — parse error: \d is not a recognised escape sequence
# RegexMatch(text, "\d+")
# RIGHT:
RegexMatch(text, "[0-9]+")
```

### 15. return/fail message must be on the same line
```jyro
# WRONG:
return
"message"    # This is a separate statement

# RIGHT:
return "message"
```

### 16. For loop bounds are exclusive
```jyro
for i in 0 to 5 do     # 0, 1, 2, 3, 4 (NOT 5)
for i in 5 downto 0 do # 5, 4, 3, 2, 1 (NOT 0)
```

### 17. Function names are PascalCase and case-sensitive
```jyro
# WRONG: toupper("hello"), toUpper("hello")
# RIGHT:
ToUpper("hello")
```

### 18. No semicolons — statements are newline-separated

### 19. `fail` is for business logic, not programming errors
Use `fail` to signal validation failures and rule violations, not for debugging.

---

## 22. Debugging Strategies

### Add intermediate variables
```jyro
# Hard to debug
Data.result = SortByField(WhereByField(Data.items, "active", "==", true), "price", "desc")

# Easy to debug
var active = WhereByField(Data.items, "active", "==", true)
Data.debugActive = active
var sorted = SortByField(active, "price", "desc")
Data.debugSorted = sorted
Data.result = sorted
```

### Inspect types and values
```jyro
Data.debug = {
    "inputType": TypeOf(Data.input),
    "inputValue": Data.input,
    "isNull": Data.input is null,
    "length": Data.input is array ? Length(Data.input) : "N/A"
}
```

### Use type guards liberally
```jyro
foreach item in Data.items do
    if item is not object then
        continue
    end
    if item.price is not number then
        continue
    end
    # Safe to process
    total += item.price
end
```

### Test with minimal data first
Start simple, then add complexity:
```json
{"items": [{"id": 1, "price": 10}]}
```
Then add edge cases:
```json
{"items": [{"id": 1, "price": 10}, {"id": 2, "price": null}, {"id": 3}]}
```

---

## 23. Complete Example

```jyro
# Order processing script
# Input: Data.orders (array of order objects)
# Output: Data.summary, Data.processed, Data.errors

# 1. Validate input
if Data.orders is null or Data.orders is not array then
    fail "Orders must be an array"
end

if Length(Data.orders) == 0 then
    Data.summary = {count: 0, total: 0}
    Data.processed = []
    return "No orders to process"
end

# 2. Quick validation
if AnyByField(Data.orders, "amount", "<", 0) then
    fail "Negative order amounts are not allowed"
end

# 3. Process orders
Data.errors = []
var processed = []
var total = 0

foreach order in Data.orders do
    if order is not object then
        Data.errors = Append(Data.errors, "Skipped non-object item")
        continue
    end

    if order.amount is not number then
        Data.errors = Append(Data.errors, "Missing amount for order: " + order.id)
        continue
    end

    total += order.amount
    order.processedAt = Now()
    order.tax = order.amount * 0.1
    order.total = order.amount + order.tax
    processed = Append(processed, order)
end

# 4. Generate summary
var count = Length(processed)
Data.summary = {
    "count": count,
    "total": total,
    "average": count > 0 ? total / count : 0,
    "errorCount": Length(Data.errors)
}

# 5. Sort and store results
Data.processed = SortByField(processed, "total", "desc")

# 6. Create export view
Data.export = Project(Data.processed, ["id", "amount", "total", "processedAt"])

return "Processed " + count + " orders"
```

---

## 24. Quick Reference: ALWAYS / NEVER

**ALWAYS:**
1. Read all input from `Data`, write all output to `Data`
2. Check for null before accessing nested properties
3. Validate divisors before division or modulo
4. Check array length explicitly — empty arrays are truthy
5. Use short-circuit `and` to guard null access: `x is not null and x.prop`
6. Create intermediate objects for nested property assignment
7. Capture return values from array functions — they return new arrays
8. Use `fail` with a message for validation errors
9. Put `return`/`fail` messages on the same line as the keyword
10. Use `WhereByField`/`FindByField` for simple field comparisons, `Where`/`Find` for complex logic
11. Pass arrays to `Sum`, `Min`, `Max`, `Average`, `Merge`, `Coalesce`
12. Use `Select` to extract field values before aggregation

**NEVER:**
1. Declare a variable named `Data`
2. Modify an array while iterating over it with `foreach`
3. Assume `ToNumber` returns 0 on invalid input — it returns null
4. Use `\d`, `\w`, `\s` in regex — they cause **parse errors**; use `[0-9]`, `[a-zA-Z0-9_]`, `[ \t\n\r]`
5. Rely on empty arrays/objects being falsy — they are truthy
6. Forget to capture return values from immutable functions (`Sort`, `Append`, etc.)
7. Forget `then` after `if`/`elseif`/`case`/`default`, `do` after `while`/`for`/`foreach`/`switch`, or `end` to close compound statements
8. Use string interpolation — it doesn't exist; use `+` concatenation
9. Pass multiple arguments to `Sum`, `Min`, `Max`, `Merge` — they take arrays
10. Use `GroupBy` with a lambda — it takes a field name string
11. Expect `Merge` to deep-merge nested objects — it's shallow
12. Put `return`/`fail` message on a different line from the keyword
