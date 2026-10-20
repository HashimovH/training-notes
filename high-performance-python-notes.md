# High Performance Python — Notes

Book: *High Performance Python* (Micha Gorelick & Ian Ozsvald)
Example code: [Mikma03/High-performance-Python](https://github.com/Mikma03/High-performance-Python) ([diffusion_2d example](https://github.com/Mikma03/High-performance-Python/blob/main/06.%20Chapter_6/diffusion_2d/README.md))

## Table of Contents

1. [Understanding Performant Python](#chapter-1--understanding-performant-python)
2. [Profiling](#chapter-2--profiling)
3. [Lists and Tuples](#chapter-3--lists-and-tuples)
4. [Dictionaries and Sets](#chapter-4--dictionaries-and-sets)
5. [Iterators and Generators](#chapter-5--iterators-and-generators)
6. [Matrix and Vector Computation](#chapter-6--matrix-and-vector-computation)
7. [Compiling to C](#chapter-7--compiling-to-c)
8. [Asynchronous I/O](#chapter-8--asynchronous-io)
9. [The multiprocessing Module](#chapter-9--the-multiprocessing-module)
10. [Clusters and Job Queues](#chapter-10--clusters-and-job-queues)
11. [Using Less RAM](#chapter-11--using-less-ram)
12. [Lessons from the Field](#chapter-12--lessons-from-the-field)

---

## Chapter 1 — Understanding Performant Python

**Topics**
- **Computing units:** CPU, GPU
- **Memory units:** L1 / L2 cache, RAM, disk. Each layer is bigger but slower than the one above it.
- **Communication layers:** buses that move data between computing and memory units. Moving data is often the real bottleneck.
- **Python virtual machine:** Python is compiled to bytecode, which the VM then interprets.
- **GIL (Global Interpreter Lock):** only one thread runs Python bytecode at a time, so threads don't speed up CPU-bound work.

**TODO:** read about CPU caches and L1/L2 in more detail.

---

## Chapter 2 — Profiling

📺 [Video](https://www.youtube.com/watch?v=VFJM4R1gFqQ)

**Topics**
- **Julia set:** the book's CPU-bound example used to demo profiling
- **CPU-bound operations:** limited by computation speed, not I/O
- **Profiling tools:**
  - `cProfile`: built-in, reports time per *function*
  - `line_profiler`: reports time per *line* (use `@profile` + `kernprof`)

**Key idea:** measure first, then optimize. Don't guess where the bottleneck is.

---

## Chapter 3 — Lists and Tuples

📺 [Video](https://www.youtube.com/watch?v=dtRHH1ITZT0)

**Topics**

| | List | Tuple |
|---|---|---|
| Mutability | Mutable (dynamic array) | Immutable (static array) |
| Memory | Over-allocates extra space | Exact size, lighter |
| Creation speed | Slower | Faster (small tuples are cached and reused by the runtime) |

- **Over-allocation:** when a list grows, Python reserves extra slots so that future `append`s don't need a resize every time. That makes `append` fast (amortized O(1)) but uses more memory.

---

## Chapter 4 — Dictionaries and Sets

📚 Reading:
- [O'Reilly, Ch. 4 (1st edition)](https://www.oreilly.com/library/view/high-performance-python/9781449361747/ch04.html)
- [Medium: High Performance Python Part 2, Sets and Dictionaries](https://medium.com/@ninads79shukla/high-performance-python-part-2-sets-and-dictionaries-e28fd1c98366)

**Topics**
- Both are **hash tables** with O(1) average lookup, insert, and delete.
- **Set** = keys only (no values). Useful for uniqueness and set operations (union, intersection).
- Performance depends on the **hash function**: collisions slow lookups down.
- **Dictionary namespaces:** variable lookups are dict lookups (`locals()` → `globals()` → `__builtins__`). Accessing globals or module attributes inside a hot loop is slower than using a local variable.

---

## Chapter 5 — Iterators and Generators

📺 [Video](https://www.youtube.com/watch?v=8rGIo5jOKRw)

**Topics**
- **Iterators & generators:** produce values one at a time with `yield`
- **How generators save memory:** values aren't all stored in a list. Only the current state is kept.
- **Lazy evaluation:** a value is computed only when it's requested, so you can work with huge or infinite sequences.

```python
# List: builds all 1M items in memory
sum([i * i for i in range(1_000_000)])

# Generator: one item at a time
sum(i * i for i in range(1_000_000))
```

---

## Chapter 6 — Matrix and Vector Computation

📚 Example: [diffusion_2d](https://github.com/Mikma03/High-performance-Python/blob/main/06.%20Chapter_6/diffusion_2d/README.md)

**Topics**
- **Cache misses & page faults**: what happens when data isn't where the CPU expects it
  - [StackOverflow: cache miss vs TLB miss vs page fault](https://stackoverflow.com/questions/37825859/cache-miss-a-tlb-miss-and-page-fault)
  - [Medium: Caching in Python](https://medium.com/@fyattani/caching-in-python-261564d64a4e)
  - [Medium: Page faults for modern developers](https://medium.com/@Tom1212121/page-faults-for-modern-developers-bf8f5c3f1f02)
- **Matrices & vectors**
  - 📺 [Video 1](https://www.youtube.com/watch?v=EP2ghkO0lSk)
  - 📺 [Video 2](https://www.youtube.com/watch?v=FmD1S5yP_os)
- **Diffusion problem**: the book's running example
  - 📺 [Video](https://www.youtube.com/watch?v=V5w5BUSiK7o)
  - 📚 [The diffusion equation (blog)](https://zsylvester.github.io/post/diffusion_equation/)

**Key ideas**
- Python lists store *pointers* to objects scattered around memory, which causes cache misses. NumPy arrays store contiguous raw data, so they're cache-friendly and can be **vectorized**.
- Avoid allocating new arrays inside loops. Reuse memory with in-place operations (`+=`, `out=`).

---

## Chapter 7 — Compiling to C

**Compilers & runtimes**
- **CPython**: the standard Python interpreter
- **Cython**: compiles Python-like code (with optional type annotations) to C
  - **pyximport**: compiles `.pyx` files automatically on import, no `setup.py` needed
- **Numba**: a **JIT** (just-in-time) compiler that uses **LLVM**. Add the `@jit` decorator to compile a function.
- **PyPy**: an alternative Python interpreter with a built-in JIT
- **GCC / g++**: the GNU C/C++ compiler toolset (used by Cython)

**Parallelism**
- **GIL**: compiled code can release the GIL to run in parallel
- **OpenMP** (Open Multi-Processing): parallelizes loops across CPU cores (e.g. `prange` in Cython)

**Other topics**
- **Garbage collection differences**: CPython uses reference counting, while PyPy uses a different GC, so objects may be freed later
- **GPU profiling**: `nvidia-smi` shows GPU usage and memory

**Foreign function interfaces** (calling existing C/Fortran code)
- **ctypes**: built-in, low-level
- **cffi**: a cleaner C interface, works well with PyPy
- **f2py**: wraps Fortran code (ships with NumPy)

---

## Chapter 8 — Asynchronous I/O

**Topics:** async programming, `async`/`await`, gevent, tornado, aiohttp, batched results

| Library | Style | Notes |
|---|---|---|
| **requests** | Serial | Simplest option. One request at a time. |
| **gevent** | Async (greenlets) | Event loop runs **only while `wait()` is running** |
| **tornado** | Async | Event loop runs **the entire time** and controls the whole execution flow |
| **aiohttp** | Async (`asyncio`) | More control over session handling and other details than tornado |

**gevent details**
- A **greenlet** is a type of coroutine. You can think of it as a lightweight thread.
- The `wait` function starts the event loop and runs it until all greenlets finish.
- **Semaphore**: limits how many coroutines can enter a block at once. Example: allow only 100 concurrent HTTP requests so you don't run out of open files or sockets.

**Batched results**: group work (e.g. DB writes) into batches so that I/O waits overlap instead of happening one by one.

---

## Chapter 9 — The multiprocessing Module

**Topics**
- **multiprocessing**: runs separate processes to get around the GIL and parallelize **CPU-bound** tasks
- **Joblib**: a simpler API for parallel loops (`Parallel`, `delayed`)
  - **Intelligent caching** with `joblib.Memory`: caches function results on disk so repeated calls are skipped
- **Multiprocessing queues**: pass work and results between processes
- **File locking**: prevents several processes from writing the same file at once

---

## Chapter 10 — Clusters and Job Queues

**Topics**
- **Clusters**: spread work across multiple machines
- **IPython Parallel**: parallel computing from IPython/Jupyter
- **Dask**: parallel Pandas-like DataFrames that scale beyond memory or a single machine
- **NSQ**: a distributed message queue
- **Pub/sub**: publishers send messages to topics and subscribers consume them

---

## Chapter 11 — Using Less RAM

**Topics**
- **Bloom filters**: probabilistic set membership. Can say "definitely not in the set" or "probably in the set", using very little memory.
- **Tries & DAWGs**: compact structures for storing many strings that share prefixes (and suffixes, in a DAWG)
- **Vectorizing text**:
  - `DictVectorizer`: maps features to columns using a stored dictionary
  - **Feature hashing** (`FeatureHasher`): hashes features straight to columns with no dictionary, which saves memory

---

## Chapter 12 — Lessons from the Field

_No notes yet._
