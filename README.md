<!---
{
  "id": "0aecd541-54e5-4f8b-8360-aeea231d05e9",
  "teaches": "C Libraries: Header-Only Libraries and Include Guards",
  "depends_on": [
    "a29aa0d7-e54c-4927-a4cc-0cd84f3b1032"
  ],
  "author": "Stephan Bökelmann",
  "first_used": "2026-10-01",
  "keywords": [
    "C",
    "header-only",
    "include guard",
    "static inline",
    "multiple definition",
    "nm"
  ]
}
--->

# C Libraries: Header-Only Libraries and Include Guards

> In this exercise you will write a small header-only library for 2D vectors, use it from two translation units, and run into the classic *multiple definition* linker error. You will then fix it with an include guard and `static inline`, and verify your understanding with `nm`.

## Introduction

A **library** is a collection of functions that several programs can reuse. The simplest form of a library in C is the *header-only library*: a single `.h` file that contains not just the prototypes but the full implementation. You ship one file, your users write one `#include`, done.

This simplicity has a price. Everything in the header is compiled again in **every** translation unit that includes it. As long as a program consists of one `.c` file, nobody notices. As soon as two `.c` files include the same header, the linker sees the same function twice and refuses to build the executable.

In this exercise you will provoke exactly this error, read it carefully, and then learn the two mechanisms that make header-only libraries work:

- an **include guard** (`#ifndef` / `#define` / `#endif`), which protects against including the same header twice *within one* translation unit, and
- `static inline`, which tells the compiler that each translation unit gets its own private copy of the function, so the linker never has to choose between them.

All work happens in your home directory. You only need `vim`, `gcc`, `nm` and a terminal. No `sudo` is required.

### 1.1) Further Readings and Other Sources

* [GCC Manual: An Inline Function is As Fast As a Macro](https://gcc.gnu.org/onlinedocs/gcc/Inline.html)
* [Beej's Guide to C: Header Files and Include Guards](https://beej.us/guide/bgc/html/split/multifile-projects.html)
* [Why do we need include guards? (StackOverflow)](https://stackoverflow.com/questions/1653958/why-are-ifndef-and-define-used-in-c-header-files)
* [nm(1) man page](https://man7.org/linux/man-pages/man1/nm.1.html)

## Tasks

### Task 1: Write the Header-Only Library

Create a working directory and open the header in `vim`:

```bash
mkdir -p ~/vec2 && cd ~/vec2
vim vec2.h
```

Implement three functions with the prefix `VEC2_`. Do **not** add an include guard yet.

```c
// file: vec2.h
#include <math.h>

typedef struct {
    double x;
    double y;
} vec2;

vec2 VEC2_add(vec2 a, vec2 b) {
    vec2 r = { a.x + b.x, a.y + b.y };
    return r;
}

double VEC2_dot(vec2 a, vec2 b) {
    return a.x * b.x + a.y * b.y;
}

double VEC2_len(vec2 a) {
    return sqrt(VEC2_dot(a, a));
}
```

### Task 2: Use It From One Translation Unit

Create `main.c`:

```c
// file: main.c
#include <stdio.h>
#include "vec2.h"

int main(void) {
    vec2 a = { 3.0, 4.0 };
    vec2 b = { 1.0, 2.0 };
    vec2 c = VEC2_add(a, b);
    printf("c = (%.1f, %.1f)\n", c.x, c.y);
    printf("|a| = %.1f\n", VEC2_len(a));
    return 0;
}
```

Compile and run:

```bash
gcc -Wall -Wextra main.c -o main -lm
./main
```

Note the `-lm`: `sqrt` lives in the math library, which is not linked by default.

### Task 3: Provoke the Multiple Definition Error

Now add a second translation unit that also uses the library:

```c
// file: util.c
#include <stdio.h>
#include "vec2.h"

void print_dot(vec2 a, vec2 b) {
    printf("a . b = %.1f\n", VEC2_dot(a, b));
}
```

Declare it in `main.c` by adding `void print_dot(vec2 a, vec2 b);` above `main` and call it. Then compile both files together:

```bash
gcc -Wall -Wextra main.c util.c -o main -lm
```

Read the error message from top to bottom. Which tool complains, the compiler or the linker? Which symbols are reported? In which object files were they found?

### Task 4: Include the Header Twice in One File

Before fixing the real problem, create a second, smaller one. In `main.c`, include `vec2.h` **twice**:

```c
#include "vec2.h"
#include "vec2.h"
```

Compile only `main.c`:

```bash
gcc -Wall -Wextra -c main.c
```

Compare this error with the one from Task 3. This time, who complains?

### Task 5: Add an Include Guard

Open `vec2.h` and wrap the whole file:

```c
#ifndef VEC2_H
#define VEC2_H

/* ... everything from before ... */

#endif /* VEC2_H */
```

Run `gcc -c main.c` again. The double include is now harmless. Then try the full build from Task 3 again:

```bash
gcc -Wall -Wextra main.c util.c -o main -lm
```

Does the include guard fix the multiple definition error? Explain why or why not before reading on.

### Task 6: Make the Functions `static inline`

Change every function definition in `vec2.h` from

```c
vec2 VEC2_add(vec2 a, vec2 b) { ... }
```

to

```c
static inline vec2 VEC2_add(vec2 a, vec2 b) { ... }
```

Build and run:

```bash
gcc -Wall -Wextra main.c util.c -o main -lm
./main
```

### Task 7: Look at the Symbol Tables

Compile the two files to object files and inspect them with `nm`:

```bash
gcc -c main.c util.c
nm main.o
nm util.o
```

Look for `VEC2_add`, `VEC2_dot` and `VEC2_len`. Which letters appear in front of them (`T`, `t`, `U`, ...)? Run `man nm` and look up what these letters mean. Then remove `static` again, rebuild the object files, and compare the output of `nm`.

## Questions

- Why did the error in Task 3 come from the linker, while the error in Task 4 came from the compiler?
- Why does an include guard not help against the multiple definition error across translation units?
- What does `static` mean for a function at file scope? What does `inline` add on top?
- In the output of `nm`, what is the difference between `T VEC2_add` and `t VEC2_add`?
- Which of the three functions was not reported by the linker in Task 3 if you never called it? Why could it still be a problem?
- Why is `-lm` needed for `sqrt` but not for `printf`?

## Advice

Header-only libraries are the easiest way to share code, and they are everywhere in the C and C++ world. But they only work if the author understands what the preprocessor does: the header is pasted, verbatim, into every file that includes it. The include guard protects against pasting twice into the same file. `static inline` protects against pasting into many files. You need both.

Keep the `VEC2_` prefix habit. The linker has one flat namespace for all functions in your program, and a function called `add` will sooner or later collide with someone else's `add`. The prefix is your namespace.

In the next exercise you will take the same library, move the implementation out of the header, and compile it separately into an archive. That removes the recompile-on-every-include cost and hides your implementation from your users.
