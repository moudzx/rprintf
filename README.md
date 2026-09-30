# rprintf

Rust-style string interpolation for C -- use `{variable}` directly in format strings with automatic type detection via `_Generic`.

```c
let(age, 26);
let(name, "Moss");

rprintf("hello {name}, you are {age} years old\n");
```

<img width="3072" height="3172" alt="Screenshot 2026-09-30 at 12-33-18 rprintf demo" src="https://github.com/user-attachments/assets/d07d536c-c30a-48c0-b2bf-611e60457429" />


## Creating and Tracking Variables

C variable names are erased by the compiler. After compilation, a variable is just a stack offset like `[rbp-4]`. There is no runtime table mapping names to values unless you build one explicitly. To bridge this gap, `rprintf` provides two macros that use the `#` preprocessor stringification operator to capture variable names and snapshot their values into a central registry:

* **`let(var, value)`**: Declares a new variable using `__typeof__`, initializes it, and immediately registers it for interpolation.
* **`watch(var)`**: Captures or updates an *already existing* variable in the registry. This must be called manually after mutating a variable to synchronize its new value with `rprintf`.

```c
let(score, 100); // Declared and automatically registered

score = 150;     // Mutated
watch(score);    // Manually updated in the registry
```

### Why `watch` is Used After Array Declarations

When working with local arrays, you must call `watch` immediately after their declaration:

```c
int scores[] = { 10, 20, 30, 0 };
watch(scores); // Snapshots the array into the registry
```

This sequence is mandatory because:
1. **Scope and Existence**: `watch` does not allocate or declare memory. It requires the variable to already be fully declared and initialized in scope so it can extract its stringified name (`"scores"`) and evaluate its address.
2. **Alternative Initialization**: Alternatively, arrays can be declared and automatically tracked in a single statement using the `let` macro with a compound literal:
   ```c
   let(scores, ((int[]){ 10, 20, 30, 0 }));
   ```

## How It Works

* **Type Detection**: `_Generic` dispatches each variable to the correct `make_*` constructor at compile time, capturing both the value and its type tag into an `Arg` tagged union.
* **Registry**: A static `VarRegistry` stores up to 256 `(name, Arg)` pairs. The macros snapshot the current value on each call.
* **Interpolation**: `rprintf` walks the format string, extracts names between `{` and `}`, looks them up in the registry, and prints them via `print_arg`.

## Limitations

* **Array Pointer Decay & Sentinel Values**: In C, passing an array to a function causes it to "decay" into a pointer, losing its compile-time size information. Because `MAKE_ARG` passes the array to functions like `make_int_arr(const int *x)`, the array's size cannot be determined via `sizeof`. Instead, arrays must be explicitly terminated with a `0` or `0.0` sentinel value so that their lengths can be dynamically scanned and calculated at runtime.
* **Manual Synchronization**: `watch` must be called manually after every mutation to update the snapshot in the registry.
* **Fixed Scale**: The registry is global, capped at 256 entries, and is not thread-safe.
* **Supported Array Types**: Native support is limited to `int[]` and `double[]`.
