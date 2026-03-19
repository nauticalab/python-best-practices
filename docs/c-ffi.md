# Python and C FFI Guide

A concise guide to calling C libraries from Python using `ctypes` and `cffi`, with practical advice on memory management and tool selection.

---

## Table of Contents

1. [Why Call C from Python?](#1-why-call-c-from-python)
2. [ctypes Basics](#2-ctypes-basics)
3. [cffi — ABI Mode](#3-cffi--abi-mode)
4. [cffi — API Mode](#4-cffi--api-mode)
5. [Choosing the Right Tool](#5-choosing-the-right-tool)
6. [Memory Management Pitfalls](#6-memory-management-pitfalls)
7. [Worked Example](#7-worked-example)

---

## 1. Why Call C from Python?

Python is expressive and productive, but sometimes you need to reach beyond its standard library:

- **Existing C libraries** — OpenSSL, libpng, BLAS, and thousands of others already exist and are battle-tested. Rewriting them in Python is wasteful; wrapping them is fast.
- **Performance-critical inner loops** — A small, hot routine (signal processing, compression, image manipulation) can be offloaded to C for a significant speed gain without rewriting the whole application.
- **Hardware access** — Device drivers, embedded SDKs, and OS-level APIs are typically C.
- **Legacy codebases** — Many organisations have decades-old C or C++ code they can't or won't rewrite. FFI lets you call it from modern Python.

Python offers three main paths into C code:

| Approach | What it is |
|---|---|
| `ctypes` | Standard library; loads shared libraries at runtime with no compilation step. |
| `cffi` | Third-party; can work at runtime (ABI mode) or with a compile step (API mode). |
| C Extension module | Write a `.c` file against the CPython C API; compiled into a `.so`/`.pyd`. |

This guide covers `ctypes` and `cffi`. C extension modules (and higher-level tools such as Cython and pybind11) are out of scope but are mentioned in [§5](#5-choosing-the-right-tool).

---

## 2. ctypes Basics

`ctypes` ships with CPython — no `pip install` needed. It loads a shared library at runtime and lets you call exported C functions directly.

### Loading a shared library

```python
import ctypes

# Linux / macOS — load by path or library name
libm = ctypes.CDLL("libm.so.6")        # Linux
libm = ctypes.CDLL("libm.dylib")       # macOS

# Windows — use WinDLL for stdcall, CDLL for cdecl
kernel32 = ctypes.WinDLL("kernel32")   # Windows only

# Portable helper that searches the system library path
import ctypes.util
libm = ctypes.CDLL(ctypes.util.find_library("m"))
```

### Defining argtypes and restype

Without type annotations ctypes assumes every argument and return value is a C `int`. This is almost always wrong and can cause segfaults or silent data corruption. **Always declare types.**

```python
import ctypes
import ctypes.util

libm = ctypes.CDLL(ctypes.util.find_library("m"))

# double sin(double x)
libm.sin.argtypes = [ctypes.c_double]
libm.sin.restype  = ctypes.c_double

result = libm.sin(1.5707963)   # ≈ 1.0
print(result)
```

Common `ctypes` types:

| C type | ctypes type |
|---|---|
| `int` | `ctypes.c_int` |
| `unsigned int` | `ctypes.c_uint` |
| `long` | `ctypes.c_long` |
| `float` | `ctypes.c_float` |
| `double` | `ctypes.c_double` |
| `char *` | `ctypes.c_char_p` |
| `void *` | `ctypes.c_void_p` |
| `size_t` | `ctypes.c_size_t` |

### Structs and pointers

```python
import ctypes

class Point(ctypes.Structure):
    _fields_ = [
        ("x", ctypes.c_double),
        ("y", ctypes.c_double),
    ]

p = Point(x=1.0, y=2.0)
ptr = ctypes.pointer(p)   # Point *
print(ptr.contents.x)     # 1.0

# Pass a pointer to a function
# void scale_point(Point *p, double factor)
mylib.scale_point.argtypes = [ctypes.POINTER(Point), ctypes.c_double]
mylib.scale_point.restype  = None
mylib.scale_point(ptr, 2.0)
```

### Passing strings and buffers

```python
import ctypes

# Immutable bytes → c_char_p
data = b"hello"
mylib.process_string.argtypes = [ctypes.c_char_p]
mylib.process_string(data)

# Writable buffer — use create_string_buffer
buf = ctypes.create_string_buffer(256)          # 256-byte zeroed buffer
mylib.fill_buffer.argtypes = [ctypes.c_char_p, ctypes.c_int]
mylib.fill_buffer(buf, ctypes.sizeof(buf))
result = buf.value    # bytes up to the first null byte
```

> **Warning:** `c_char_p` passes the address of the Python bytes object's internal buffer. Never store that pointer on the C side longer than the call lasts — the bytes object may be garbage-collected.

---

## 3. cffi — ABI Mode

[cffi](https://cffi.readthedocs.io/) takes a different approach: you give it a C header (as a string), and it parses it to generate a clean Python binding. The result is faster than `ctypes` for repeated calls and produces more Pythonic error messages.

**Install:** `pip install cffi`

### Inline ABI mode

Inline ABI mode is the simplest form — everything happens at runtime, no compilation needed. Use it for quick scripts or when distributing source-only packages.

```python
from cffi import FFI

ffi = FFI()

# Paste (a subset of) the C declarations you need
ffi.cdef("""
    double sin(double x);
    double cos(double x);
""")

# Open the shared library
libm = ffi.dlopen("libm.so.6")   # or ffi.dlopen(None) for the C runtime on Windows

result = libm.sin(1.5707963)
print(result)   # ≈ 1.0
```

`ffi.cdef` accepts standard C declarations — structs, typedefs, enums, and function prototypes. You do **not** need to include the full header; declare only what you use.

### Out-of-line ABI mode

Out-of-line mode moves the `ffi.cdef` call into a separate build module so parsing happens once at install time, not on every import.

```python
# _libm_build.py  (run once: python _libm_build.py)
from cffi import FFI

ffi = FFI()
ffi.cdef("""
    double sin(double x);
    double cos(double x);
""")
ffi.set_source(
    "_libm",           # name of the generated extension module
    None,              # None = ABI mode (no C compilation)
    libraries=["m"],   # link against libm
)

if __name__ == "__main__":
    ffi.compile(verbose=True)
```

```python
# usage.py
from _libm import ffi, lib

print(lib.sin(1.5707963))
```

---

## 4. cffi — API Mode

API mode compiles a small C wrapper at install time. The compiler checks your declarations against the actual header, catching mismatches before they become runtime surprises. This is the recommended mode for production packages.

```python
# _mylib_build.py
from cffi import FFI

ffi = FFI()

# Declare only the functions/types you use
ffi.cdef("""
    typedef struct {
        double x;
        double y;
    } Point;

    Point make_point(double x, double y);
    double distance(Point a, Point b);
""")

ffi.set_source(
    "_mylib",
    """
    #include "mylib.h"   /* actual C header — the compiler validates types */
    """,
    sources=["mylib.c"],   # C source files to compile
    # libraries=["m"],     # additional libraries to link
)

if __name__ == "__main__":
    ffi.compile(verbose=True)
```

```python
# usage.py
from _mylib import ffi, lib

p = lib.make_point(3.0, 4.0)
q = lib.make_point(0.0, 0.0)
print(lib.distance(p, q))   # 5.0
```

The key difference from ABI mode: `set_source` receives a real C snippet (not `None`). The snippet is compiled by the system C compiler, so types are resolved from the actual header rather than your hand-written declarations.

> **Tip:** Add the build script to your `pyproject.toml` as a `cffi_modules` entry (via `setuptools` + `cffi`) so it runs automatically during `pip install`.

---

## 5. Choosing the Right Tool

| | `ctypes` | `cffi` ABI | `cffi` API | C extension |
|---|---|---|---|---|
| **Compilation needed** | No | No | Yes | Yes |
| **Type safety** | Manual | Manual | Compiler-verified | Full |
| **Performance** | Good | Better | Best (comparable to native) | Best |
| **Debugging** | Hard | Moderate | Moderate | Best (full debugger support) |
| **Portability** | Excellent | Excellent | Good | Good |
| **Complexity** | Low | Low | Medium | High |
| **Best for** | Quick scripts, stdlib-only | Distributing source packages | Production packages | Complex APIs, maximum speed |

**Rough decision guide:**

- **Reach for `ctypes`** when you need a one-off call to a system library, you cannot add dependencies, or you're writing a quick utility script.
- **Reach for `cffi` ABI mode** when `ctypes` becomes unwieldy (many structs, callbacks) but you don't want to require a C compiler on end-user machines.
- **Reach for `cffi` API mode** when you're shipping a package and want the compiler to catch declaration mismatches at build time.
- **Write a C extension / use Cython / use pybind11** when you need to implement new logic in C/C++ (not just wrap it), expose CPython internals, or squeeze out every last microsecond of performance.

---

## 6. Memory Management Pitfalls

Crossing the Python/C boundary means crossing memory ownership boundaries. Most FFI bugs fall into a small number of patterns.

### Who owns the pointer?

The most important question for any pointer returned from a C function is: *who is responsible for freeing it?*

```python
# Pattern 1: Python allocates, C reads — Python owns
buf = ctypes.create_string_buffer(1024)
mylib.fill(buf, 1024)
# buf is freed when it goes out of scope

# Pattern 2: C allocates, caller must free — you own
ptr = mylib.create_object()          # returns c_void_p
# ... use ptr ...
mylib.destroy_object(ptr)            # must call manually
```

Forgetting to call the C free function leaks memory. Calling it twice causes a double-free (usually a crash or heap corruption).

### Keeping Python objects alive

Python's garbage collector doesn't know about references held by C code. If you pass a Python object's address to C and the Python side drops all references to that object, it can be collected while C is still using it.

```python
import ctypes

def bad():
    data = b"temporary"
    ptr = ctypes.cast(data, ctypes.c_char_p)
    # data may be collected here if the compiler optimises away the reference
    mylib.use_string(ptr)    # use-after-free!

def good():
    data = b"temporary"
    mylib.use_string(data)   # pass the object directly; ctypes pins it for the call
    # or keep `data` alive explicitly until C is done
```

Use `ctypes.byref()` or `ctypes.pointer()` only for the duration of the call. Never store raw ctypes pointers and use them after the Python object they point into has been freed.

### cffi: using `ffi.gc()` for automatic cleanup

```python
# Let cffi call your destructor automatically when the Python wrapper is GC'd
raw_ptr = lib.create_object()
managed = ffi.gc(raw_ptr, lib.destroy_object)
# managed behaves like raw_ptr but calls lib.destroy_object(managed) on collection
```

`ffi.gc` is analogous to `std::unique_ptr` in C++ — ownership is tied to the Python object's lifetime.

### Buffer protocol and zero-copy I/O

When passing large binary data, avoid unnecessary copies:

```python
import ctypes

data = bytearray(1024 * 1024)   # 1 MiB

# ctypes can take a memoryview/bytearray without copying
ptr = (ctypes.c_char * len(data)).from_buffer(data)
mylib.process(ptr, len(data))
# `data` must stay alive for the duration of the call
```

`from_buffer` gives you a ctypes array that shares memory with the Python buffer — no copy, but the original buffer must remain alive and not be resized while C is reading it.

### Common pitfalls summary

| Pitfall | Symptom | Fix |
|---|---|---|
| No `argtypes`/`restype` | Segfault, wrong results | Always declare types |
| Pointer outlives Python object | Use-after-free, random crashes | Keep owning object alive |
| Missing `ffi.gc` / manual free | Memory leak | Use `ffi.gc` or explicit free |
| Double-free | Heap corruption, crash | Track ownership clearly |
| Resizing buffer under C | Heap corruption | Freeze the buffer while C uses it |

---

## 7. Worked Example

The following example wraps a small hypothetical C library that computes a running statistics summary (mean and variance) over a stream of `double` values. It demonstrates `cffi` API mode end-to-end — from the C source to Python usage.

### C source (`stats.h` and `stats.c`)

```c
/* stats.h */
#ifndef STATS_H
#define STATS_H

typedef struct Stats Stats;

Stats  *stats_new(void);
void    stats_free(Stats *s);
void    stats_update(Stats *s, double value);
size_t  stats_count(const Stats *s);
double  stats_mean(const Stats *s);
double  stats_variance(const Stats *s);   /* population variance */

#endif
```

```c
/* stats.c */
#include <stdlib.h>
#include "stats.h"

struct Stats {
    size_t n;
    double mean;
    double M2;   /* Welford's algorithm accumulator */
};

Stats *stats_new(void) {
    Stats *s = calloc(1, sizeof(Stats));
    return s;
}

void stats_free(Stats *s) { free(s); }

void stats_update(Stats *s, double x) {
    s->n++;
    double delta  = x - s->mean;
    s->mean      += delta / s->n;
    double delta2 = x - s->mean;
    s->M2        += delta * delta2;
}

size_t stats_count(const Stats *s) { return s->n; }
double stats_mean(const Stats *s)  { return s->mean; }
double stats_variance(const Stats *s) {
    return s->n < 2 ? 0.0 : s->M2 / s->n;
}
```

### cffi build script (`_stats_build.py`)

```python
# _stats_build.py
from cffi import FFI

ffi = FFI()

ffi.cdef("""
    typedef struct Stats Stats;

    Stats  *stats_new(void);
    void    stats_free(Stats *s);
    void    stats_update(Stats *s, double value);
    size_t  stats_count(const Stats *s);
    double  stats_mean(const Stats *s);
    double  stats_variance(const Stats *s);
""")

ffi.set_source(
    "_stats",
    '#include "stats.h"',
    sources=["stats.c"],
)

if __name__ == "__main__":
    ffi.compile(verbose=True)
```

Run once to compile:

```bash
python _stats_build.py
```

### Python usage

```python
from _stats import ffi, lib


def compute_stats(values):
    """Return (count, mean, variance) for an iterable of floats."""
    s = ffi.gc(lib.stats_new(), lib.stats_free)  # auto-freed on GC
    if s == ffi.NULL:
        raise MemoryError("stats_new returned NULL")

    for v in values:
        lib.stats_update(s, v)

    return lib.stats_count(s), lib.stats_mean(s), lib.stats_variance(s)


data = [2.0, 4.0, 4.0, 4.0, 5.0, 5.0, 7.0, 9.0]
count, mean, var = compute_stats(data)
print(f"n={count}, mean={mean:.2f}, variance={var:.2f}")
# n=8, mean=5.00, variance=4.00
```

Key points from this example:

- `ffi.gc(lib.stats_new(), lib.stats_free)` ties the C object's lifetime to the Python variable `s` — no manual `stats_free` call needed.
- The `ffi.NULL` check mirrors the `if (!s) return NULL;` pattern you'd write in C.
- The caller never sees a raw C pointer; `s` behaves like a Python object.
- Because `values` is iterated once in Python and each `double` is passed by value, there are no ownership ambiguities.

---

## Further Reading

- [`ctypes` — Python documentation](https://docs.python.org/3/library/ctypes.html)
- [`cffi` documentation](https://cffi.readthedocs.io/)
- [Extending Python with C or C++](https://docs.python.org/3/extending/extending.html)
- [Cython](https://cython.readthedocs.io/) — write Python-like code that compiles to C
- [pybind11](https://pybind11.readthedocs.io/) — expose C++11 code to Python with minimal boilerplate
