# How to Speed Up MicroPython

# 1. Optimisation Steps

We can compare the MicroPython code optimisation process to "steps for speeding up a bicycle": first find the core reasons why the bicycle is slow (e.g., flat tyres, chain friction), then start with simple adjustments (inflating tyres, lubricating the chain), and finally consider replacing high-end parts (lightweight frame, carbon fibre wheels). MicroPython code optimisation also follows a **"simple to complex, software to hardware"** order, allowing you to achieve the greatest performance improvement at the lowest cost, avoiding wasting time by diving into complex low-level optimisation from the start.

Before starting optimisation, understand a few core foundational concepts:

The process of developing high-performance code includes the following stages, which should be executed in the order listed:

1. **Identify the slowest code**: Use timing tools to find the real performance bottleneck, rather than optimising based on intuition.
2. **Memory and object optimisation**: Reduce heap allocation, avoid dynamic object creation, and lower the frequency of GC triggers.
3. **Code execution efficiency optimisation**: Use `const()`, precompiled bytecode, native/viper code emitters.
4. **Computation and hardware optimisation**: Replace floats with integers, directly manipulate registers, use DMA.

# 2. Optimisation Methods

## 2.1 Identify the Slowest Code

To identify the slowest code, we can typically use the `utime` module's tick recording functions by judiciously using timing. Code execution time can be measured in ms (milliseconds), us (microseconds), or CPU cycles.

The following code allows timing any function or method by adding the `@timed_function` decorator:

```python
import time

def timed_function(f, *args, **kwargs):
    myname = str(f).split(' ')[1]
    def new_func(*args, **kwargs):
        t = time.ticks_us()
        result = f(*args, **kwargs)
        delta = time.ticks_diff(time.ticks_us(), t)
        print('Function {} Time = {:6.3f}ms'.format(myname, delta/1000))
        return result
    return new_func
```

Here, we use the mpremote tool to connect to a Raspberry Pi Pico, copy the above code to the REPL, and press Enter to execute (the decorator is now defined):

```
Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

Try the new cross-platform PowerShell https://aka.ms/pscore6

(base) PS D:\lee\windows terminal\terminal-1.17.11461.0> mpremote
Connected to MicroPython at COM65
Use Ctrl-] or Ctrl-x to exit this shell

>>> import time
>>>
>>> def timed_function(f, *args, **kwargs):
...         myname = str(f).split(' ')[1]
...         def new_func(*args, **kwargs):
...                 t = time.ticks_us()
...                 result = f(*args, **kwargs)
...                 delta = time.ticks_diff(time.ticks_us(), t)
...                 print('Function {} Time = {:6.3f}ms'.format(myname, delta/1000))
...                 return result
...         return new_func
...
>>>
```

Next, define a test function (simulating a time-consuming operation):

```python
# Decorate the test function with @timed_function
@timed_function
def slow_function():
    for i in range(10000):
        pass
    return "Done"
```

We paste it into the REPL:

```
>>> @timed_function
... def slow_function():
...         for i in range(10000):
...                 pass
...         return "Done"
...
>>>
```

Then call the function to see the timing result:

```python
slow_function()
```

The execution time of the function is output:

```
>>> slow_function()
Function slow_function Time =  35.915ms
'Done'
>>>
```

## 2.2 Specific Performance Optimisation Measures

### 2.2.1 MicroPython Performance Bottlenecks and Core Foundational Concepts

#### 2.2.1.1 Core Foundational Concepts

##### 2.2.1.1.1 Variables and Constants

First, we need to understand the difference between variables and constants. This is fundamental to programming. The core difference lies in when their value is determined:

| Feature | Variable | Constant |
|---------|----------|----------|
| Value determination time | Dynamically assigned at runtime | Fixed at compile/definition time |
| Access method | Runtime dictionary lookup (has overhead) | Replaced directly with value at compile time (no overhead) |
| Memory allocation | May trigger heap allocation | Does not trigger heap allocation |
| MicroPython usage | Normal variable assignment | `from micropython import const; X = const(value)` |
| Use case | Values that need to change during execution | Fixed values like register addresses, max counts, pin masks |

##### 2.2.1.1.2 Three Memory Partitions

Embedded microcontrollers (like the RP2040) have very limited memory (e.g., only 264KB RAM), and MicroPython's object storage directly depends on memory partitions. **Heap allocation and garbage collection are the biggest performance bottlenecks.** Let's use a "warehouse management" analogy to understand the three partitions:

| Partition | Analogy | Stored Content | Allocation Method | Performance Impact |
|-----------|---------|----------------|-------------------|--------------------|
| **Stack** | Workbench (limited space, use and discard) | Local variables, function call frames, return addresses | Automatic allocation/release (LIFO) | Extremely fast, no GC overhead |
| **Heap** | Warehouse (large space, needs registration to retrieve) | All Python objects (lists, dicts, bytearrays, etc.) | Dynamic allocation, GC reclamation | Slow allocation, GC trigger is time-consuming |
| **Static Area (BSS/Data)** | Fixed shelves (placed at startup, not moved) | Global variables, module-level constants, firmware code | Allocated at program startup, not released | Fast access, no runtime overhead |

For memory management in embedded development, the following difficulties are often encountered:

- **Heap memory fragmentation**: After frequent allocation/release of small objects, free memory becomes non-contiguous, making it impossible to allocate large blocks.
- **Random GC triggering**: Garbage collection can be triggered at any time, causing unpredictable delays (several milliseconds) in performance-critical code sections.
- **Out-of-memory crashes**: When heap space is exhausted, a `MemoryError` is raised at runtime, causing the program to crash.
- **Stack overflow**: When recursion is too deep or there are too many local variables, stack space is exhausted, causing a hardware exception.

##### 2.2.1.1.3 References and Copies

A reference is like an object's **"address number"** (analogy: a shortcut on a computer). The variable stores not the data itself, but the address of the data in memory. Operating on a reference does not copy the data, and the overhead is minimal.

```python
# Create a bytearray object on the heap; 'a' stores the object's address (address number)
a = bytearray(10)  
# 'b' is a reference to 'a', pointing to the same object, no new allocation
b = a  
# Operating on 'b' changes the content of 'a' because they are the same object
b[0] = 1  
# Output: 1
print(a[0])
```

In the terminal, the result is:

```
>>> a = bytearray(10)
>>> b = a
>>> b[0] = 1
>>> print(a[0])
1
>>>
```

For a copy (here referring to a deep copy; the difference between shallow and deep copy is not explained here), it copies the entire data, creates a new object on the heap, and stores a new data set. The overhead is significant (especially for large data):

```python
# Large bytearray (10KB on heap)
a = bytearray(10000)  
# Slice operation, creates a new bytearray (~2KB on heap), this is a deep copy
b = a[30:2000]
```

After execution, `b` is a brand new `bytearray` object containing a complete copy of the data from `a[30:2000]`. Printing `b` shows 1970 `\x00` bytes:

```
>>> a = bytearray(10000)
>>> b = a[30:2000]
>>> b
bytearray(b'\x00\x00\x00\x00...\x00\x00\x00\x00')
```

To avoid the large heap allocation problem when copying large bytearrays, we can use a memoryview. It is a shallow reference tool provided by MicroPython, essentially a "read-only/writable address number" for buffer objects (bytearrays, arrays, bytes, etc.). Slicing with a memoryview does not copy data, only passes the address, completely avoiding heap allocation.

```python
# Large bytearray
a = bytearray(10000)  
# Create a memoryview (only allocates a small object, tens of bytes)
mv = memoryview(a)    
# Slice the memoryview, no new allocation, only passes the address
b = mv[30:2000]       
# Operating on 'b' changes the content of 'a' because they are the same data
b[0] = 1           
print(a[30])
```

The result is:

```
>>> a = bytearray(10000)
>>> mv = memoryview(a)
>>> b = mv[30:2000]
>>> b[0] = 1
>>> print(a[0])
0
>>> print(a[30])
1
>>>
```

> **Note**: memoryview only supports objects that implement the buffer protocol (bytearray, array, bytes). It does not support lists (lists store object references, not contiguous data).

With the basics above understood, let's understand these advanced concepts for embedded MicroPython:

| Advanced Concept | Description | Performance Impact |
|------------------|-------------|--------------------|
| **GC (Garbage Collection)** | Automatically reclaims objects on the heap that are no longer referenced, freeing memory | Each GC takes 1 to tens of ms, trigger time is unpredictable |
| **Bytecode Interpretation** | MicroPython compiles `.py` to bytecode and then interprets it instruction by instruction | 10 to 100 times slower than native C code |
| **native/viper Emitter** | Compiles a function directly to ARM machine code, skipping bytecode interpretation | native is ~2x faster, viper integer operations are tens of times faster |
| **DMA (Direct Memory Access)** | Transfers data between memory and peripherals autonomously, without CPU intervention | Frees the CPU from data movement, allowing it to focus on computation |

#### 2.2.1.2 MicroPython Performance Bottlenecks

MicroPython's performance bottlenecks mainly come from three core aspects: **heap memory allocation and garbage collection (GC) overhead**, **Python bytecode interpretation overhead**, and **inefficient computation/hardware operation methods**.

Therefore, we can optimise from the following aspects:

| Optimisation Level | Main Methods | Expected Benefit |
|--------------------|--------------|------------------|
| **Memory and Object Optimisation** | Pre-allocation, fixed sizes, memoryview, caching references, manual GC | Eliminates random GC pauses, significantly improves stability |
| **Code Execution Efficiency Optimisation** | `const()`, mpy-cross precompilation, native/viper decorators | Tens of times faster integer operations, faster loading |
| **Computation and Hardware Optimisation** | Replace floats with integers, direct register access (mem32), DMA | Minimises hardware operation latency, maximises CPU resource utilisation |

### 2.2.2 Memory and Object Optimisation

The core of this type of optimisation is to **"avoid dynamically creating objects at runtime as much as possible, reduce heap allocation, and thus lower the frequency and duration of GC triggers."** This is the primary step in embedded MicroPython performance optimisation.

#### 2.2.2.1 Pre-allocate Memory and Fix Object Sizes

Create objects only once (e.g., instantiate in the class constructor), and do not allow their size to grow dynamically (e.g., list `append`, adding key-value pairs to a dictionary). Especially for buffers (e.g., serial communication buffers), pre-allocate and reuse them.

We can use `readinto()` instead of `read()` (`read()` allocates a new buffer each time, while `readinto()` reads data into an existing buffer).

Using a serial buffer as an example:

```python
from machine import UART, Pin

# 1. Pre-allocate buffer (created only once, avoids dynamic allocation)
buf = bytearray(64)  # Pre-allocate a 64-byte buffer

# Initialise UART
uart = UART(0, baudrate=9600, tx=Pin(0), rx=Pin(1))

# 2. Use readinto() to read into the pre-allocated buffer (no new allocation)
while True:
    if uart.any():
        # Read data into buf, returns the number of bytes read
        n = uart.readinto(buf)  
        # Compare: uart.read(64) would create a new bytes object each time, triggering heap allocation
        print("recv data:", buf[:n])
```

#### 2.2.2.2 Use Arrays Instead of Lists + memoryview to Avoid Data Copying

Lists store object references, memory is non-contiguous, and dynamic growth triggers heap allocation. The `array` module (or `bytearray`) stores contiguous basic type data, offering higher performance after pre-allocation. Also, slice operations (e.g., `ba[30:2000]`) create data copies, triggering heap allocation. Using `memoryview` directly passes a memory pointer with no copy overhead.

```python
import array
import time

def timed_function(f, *args, **kwargs):
    myname = str(f).split(' ')[1]
    def new_func(*args, **kwargs):
        t = time.ticks_us()
        result = f(*args, **kwargs)
        delta = time.ticks_diff(time.ticks_us(), t)
        print('Function {} Time = {:6.3f}ms'.format(myname, delta/1000))
        return result
    return new_func

# 1. Use array instead of list (stores integers, contiguous memory)
# Pre-allocate an array of 1000 int elements
arr = array.array('i', [0]*1000)  
# 2. Use memoryview to avoid slice copying
# Large bytearray
ba = bytearray(10000)  

# Direct slice: creates a copy, triggers ~2K heap allocation
@timed_function
def func(data):
    pass
    
# Test slice copy (time-consuming and memory-intensive)
func(ba[30:2000])

# Use memoryview: only allocates a small object, no data copy
mv = memoryview(ba)
# Passes a memory pointer, no allocation
func(mv[30:2000])
```

Terminal output:

```
>>> func(ba[30:2000])
Function func Time =  0.095ms
>>> mv = memoryview(ba)
>>> func(mv[30:2000])
Function func Time =  0.079ms
>>>
```

It can be seen that slicing with `memoryview` is slightly faster than direct slicing, and more importantly, **it avoids heap allocation** (the advantage is more significant with large data).

#### 2.2.2.3 Cache Object References

Cache frequently accessed objects (e.g., `self.ba`, `obj_display.framebuffer`) into local variables to avoid attribute lookup each time (attribute lookup involves dictionary operations, which are time-consuming and may trigger allocation).

```python
import time

def timed_function(f, *args, **kwargs):
    myname = str(f).split(' ')[1]
    def new_func(*args, **kwargs):
        t = time.ticks_us()
        result = f(*args, **kwargs)
        delta = time.ticks_diff(time.ticks_us(), t)
        print('Function {} Time = {:6.3f}ms'.format(myname, delta/1000))
        return result
    return new_func

class Foo:
    def __init__(self):
        self.ba = bytearray(100)  
    @timed_function
    def bar(self):
        ba_ref = self.ba          # Cache reference to local variable
        for i in range(100):
            ba_ref[i] = i % 256
            
class Foo_compare:
    def __init__(self):
        self.ba = bytearray(100)  
    @timed_function
    def bar(self):
        for i in range(100):
            self.ba[i] = i % 256  # Attribute lookup every loop
            
# Test
f = Foo()
f.bar()

f_c = Foo_compare()
f_c.bar()
```

Click run, terminal output:

```
>>> f = Foo()
>>> f.bar()
Function bar Time =  1.039ms
>>> f_c = Foo()
>>> f_c.bar()
Function bar Time =  0.999ms
```

> Caching references is significantly more effective in scenarios with large loops (e.g., 10000+ iterations) and nested attributes (e.g., `self.obj.buf`).

#### 2.2.2.4 Manual Garbage Collection Control

Periodically call `gc.collect()` to manually trigger GC, preventing GC from being triggered randomly during performance-critical code sections (manual GC allows controlling the timing, and the total time of frequent small GCs is much less than a single large GC).

```python
import gc
import time

# Enable GC (enabled by default, can be manually disabled/enabled)
gc.enable()
# Before a performance-critical loop, manually trigger GC
# Clean up memory in advance, takes about 1ms
gc.collect()  

# Performance-critical code section
start = time.ticks_us()
for i in range(10000):
    pass
end = time.ticks_us()

print(f"Time: {time.ticks_diff(end, start)/1000}ms")
# Periodically trigger GC in non-critical sections
# gc.collect()
```

### 2.2.3 Code Execution Efficiency Optimisation

This type of optimisation builds upon memory optimisation to further improve code execution speed, optimising from the bytecode level to the machine code level.

#### 2.2.3.1 Use const() to Declare Constants

`const()` replaces an identifier with its value (done at compile time), avoiding runtime dictionary lookups. The optimisation effect is significant, especially for constants used in loops.

```python
from micropython import const
import time

# Declare constants (replaced with values at compile time)
MAX_COUNT = const(100000)
# Binary constants are also supported
PIN_BIT = const(1 << 2)

MAX_COUNT_NOT_USE_CONST = 100000

def timed_function(f, *args, **kwargs):
    myname = str(f).split(' ')[1]
    def new_func(*args, **kwargs):
        t = time.ticks_us()
        result = f(*args, **kwargs)
        delta = time.ticks_diff(time.ticks_us(), t)
        print('Function {} Time = {:6.3f}ms'.format(myname, delta/1000))
        return result
    return new_func

@timed_function
def use_const():
    total = 0
    for i in range(MAX_COUNT):
        total += i
    return total

# Compare: without const(), each access requires a dictionary lookup, taking longer
@timed_function
def no_const():
    global MAX_COUNT_NOT_USE_CONST
    total = 0
    for i in range(MAX_COUNT_NOT_USE_CONST):
        total += i
    return total

result1 = use_const()
result2 = no_const()
```

Terminal result:

```
>>> result1 = use_const()
Function use_const Time = 1120.690ms
>>> result2 = no_const()
Function no_const Time = 1120.892ms
```

> **Note**: The execution times are very similar because in the REPL, the code is interpreted. The real advantage of `const` is evident in **precompiled bytecode**. When the code is saved as a `.py` file and executed via `import`, the optimisation effect of `const()` is more significant.

#### 2.2.3.2 mpy-cross Compile Bytecode

Use `mpy-cross` on your computer to precompile `.py` scripts into `.mpy` bytecode, then upload them to the device.

**Install mpy-cross**:

```
(base) PS D:\lee\windows terminal\terminal-1.17.11461.0> pip install mpy-cross
Looking in indexes: https://pypi.tuna.tsinghua.edu.cn/simple
Collecting mpy-cross
  Downloading https://pypi.tuna.tsinghua.edu.cn/packages/.../mpy_cross-1.27.0.post2-py2.py3-none-win_amd64.whl (1.2 MB)
  | 1.2 MB  1.1 MB/s
Installing collected packages: mpy-cross
Successfully installed mpy-cross-1.27.0.post2
```

**Compile `.py` to `.mpy`**:

```
(base) PS G:\test_microMLP> mpy-cross main.py
(base) PS G:\test_microMLP> mpy-cross microMLP.py
```

After compilation, you can see both `.py` and `.mpy` files in the directory. The `.mpy` files are smaller:

| File Name | Type | Size |
|-----------|------|------|
| main.mpy | MPY file | 1 KB |
| main.py | Python source file | 2 KB |
| microMLP.mpy | MPY file | 10 KB |
| microMLP.py | Python source file | 32 KB |

**Upload to the device** using the `mpremote` tool:

```
(base) PS G:\test_microMLP> mpremote fs cp main.mpy :main.mpy
cp main.mpy :main.mpy
Up to date: main.mpy
(base) PS G:\test_microMLP> mpremote fs ls
ls :
    5876 AbstractBlockDevInterface.py
     572 LICENSE
    8192 README.md
   25420 ads1115.py
   11720 bh_1750.py
   14474 dbt22.py
     800 main.mpy
    1401 main.py
    9154 sd_block_dev.py
   17702 sdcard.py
```

`.mpy` files are smaller than `.py` files (microMLP.py 32KB → microMLP.mpy 10KB), and they skip the compilation stage when loading, resulting in faster loading.

#### 2.2.3.3 Using Code Emitters

When MicroPython compiles code, it processes each function individually (classes are functions, lambdas and list comprehensions are also functions). Functions come out of the parsing stage and then enter the compiler, which passes the Python function 3 times:

Currently, there are four types of code emitters:

| Emitter | Description | Speed | Compatibility |
|---------|-------------|-------|---------------|
| **Bytecode Emitter (Default)** | Generates bytecode, interpreted execution | Baseline speed | Fully compatible with Python |
| **native Emitter** | Generates ARM-Thumb machine code, executes directly | ~2x speedup | Has limitations (see below) |
| **viper Emitter** | Generates optimised machine code, focuses on integer/pointer operations | Tens of times faster for integer operations | Not fully compatible, requires type annotations |
| **Inline Assembler Emitter** | Embeds assembly instructions directly, extreme performance | Fastest | Requires assembly knowledge, poor portability |

##### 2.2.3.3.1 native Code Emitter

The native code emitter takes each bytecode and converts it into equivalent ARM-Thumb machine code. Such functions use the normal C stack to store local variables and call C runtime functions directly.

The native code emitter is invoked via a function decorator:

```python
@micropython.native
def foo(self, arg):
    buf = self.linebuf  # Cached object
    # code
```

The current implementation of the native code emitter has certain limitations:

- Does not support generators
- Does not support keyword argument passing
- Increases compiled code size (takes up more Flash than bytecode)

**The price for improved performance (approximately twice that of bytecode) is an increase in compiled code size.**

##### 2.2.3.3.2 Viper Code Emitter

The optimisations discussed above involve standard-compliant Python code. The Viper code emitter is not fully compatible. It supports special Viper native data types to pursue performance. Integer handling is non-compliant because it uses machine words: arithmetic on 32-bit hardware is performed modulo 2\*\*32.

The Viper code emitter emits ARM-Thumb machine code for each bytecode and further optimises certain things, such as integer operations. For adding two integers, the viper emitter does not call the C runtime function `rt_binary_op`, but instead emits the machine instruction `adds` to add the two numbers directly. This is much faster than calling `rt_binary_op`. It is invoked using a decorator:

```python
@micropython.viper
def foo(self, arg: int) -> int:
    # code
```

Viper supports its own set of types: `int` (signed integer), `uint` (unsigned integer), `ptr` (generic pointer), `ptr8` (byte pointer), `ptr16` (16-bit pointer), `ptr32` (32-bit pointer).

| Type | Description | Use Case |
|------|-------------|----------|
| `int` | Signed machine word integer (32-bit) | General integer arithmetic, modulo 2^32 |
| `uint` | Unsigned machine word integer (32-bit) | Bit operations, address calculation |
| `ptr8` | Pointer to byte (uint8) | Accessing bytearray, bytes, etc. |
| `ptr16` | Pointer to 16-bit integer | Accessing array('H'), etc. |
| `ptr32` | Pointer to 32-bit integer | Direct access to register addresses |

**Test large-scale integer accumulation**:

```python
import time
import micropython

def timed_function(name):
    def decorator(f):
        def new_func(*args, **kwargs):
            t = time.ticks_us()
            result = f(*args, **kwargs)
            delta = time.ticks_diff(time.ticks_us(), t)
            print('Function {} Time = {:6.3f}ms'.format(name, delta/1000))
            return result
        return new_func
    return decorator
    
# Normal Python function: 1 million accumulations
@timed_function('normal_add_loop')
def normal_add_loop(n: int) -> int:
    total = 0
    for i in range(n):
        total += i * 2 + 5
    return total

# Viper optimised function: same calculation
@timed_function('viper_add_loop')
@micropython.viper
def viper_add_loop(n: int) -> int:
    total = 0
    for i in range(n):
        total += i * 2 + 5
    return total

# Call test (1 million operations)
normal_add_loop(1000000)
viper_add_loop(1000000)
```

Both functions perform exactly the same 1 million integer operations (`i*2+5` accumulation), but `normal_add_loop` is purely interpreted, while `viper_add_loop` executes machine code directly. Terminal result:

```
>>> normal_add_loop(1000000)
Function normal_add_loop Time = 17221.732ms
1000004000000
>>> viper_add_loop(1000000)
Function viper_add_loop Time = 296.095ms
-723379968
```

> **Result Analysis**:
> - `normal_add_loop` took **17221.732ms** (~17.2 seconds)
> - `viper_add_loop` took **296.095ms** (~0.3 seconds)
> - The Viper version is **~58 times faster**!
> - Note: The return value of `viper_add_loop` is `-723379968`. This is because Viper uses 32-bit machine word integers. The accumulation of 1 million operations exceeds the range of a 32-bit signed integer (2^31 ≈ 2.1 billion), causing an overflow truncation.

**Viper has two key limitations**:

1. **Does not support default arguments**: Parameters of Viper functions must be passed explicitly; default values are not supported.

```python
import micropython

# Correct example (no default parameters, 1 million operations, timed)
@micropython.viper
def viper_no_default(a: int) -> int:
    total = 0
    for i in range(a):
        total += i
    return total

# Call test
viper_no_default(1000000)
```

If you define a Viper function with default parameters and call it without arguments, an error occurs:

```
>>> viper_default()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: function takes 1 positional arguments but 0 were given
>>> viper_default(10)
45
```

(The Viper compiler discards the default parameter information, causing the function signature to become "requires 1 argument".)

2. **No optimisation for floating-point operations**: Viper does not speed up floating-point calculations.

```python
import micropython

# Normal function: 100k floating-point multiplication and accumulation
@timed_function('normal_float_calc')
def normal_float_calc(n: int) -> float:
    total = 0.0
    for i in range(n):
        total += float(i) * 3.14159
    return total

# Viper function: same 100k floating-point operations (no optimisation)
@timed_function('viper_float_calc')
@micropython.viper
def viper_float_calc(n: int):
    total = 0.0
    for i in range(n):
        total += float(i) * 3.14159
    return total

# Call test (timing results are almost identical)
normal_float_calc(100000)
viper_float_calc(100000)
```

Result:

```
>>> normal_float_calc(100000)
Function normal_float_calc Time = 2887.723ms
1.570779e+10
>>> viper_float_calc(100000)
Function viper_float_calc Time = 2458.806ms
1.570779e+10
>>> normal_float_calc(100000)
Function normal_float_calc Time = 2887.671ms
1.570779e+10
>>> viper_float_calc(100000)
Function viper_float_calc Time = 2458.804ms
1.570779e+10
```

> Viper's floating-point time (2458ms) is very close to the normal function's time (2887ms), with only about a 15% speedup. This is far from the 58x speedup for integer operations, indicating that **Viper is not suitable for floating-point intensive calculations**.

**Viper Pointer Types (ptr8/ptr16/ptr32) for Direct Contiguous Memory Access**:

Viper's pointer types are used for direct access to contiguous memory (e.g., `bytearray`), without bounds checking. They support single-element access via subscript (but not slicing). A key optimisation tip is: **perform the object-to-pointer conversion at the beginning of the function** (not inside the loop), because the conversion operation takes a few microseconds, which is amplified in large loops.

The advantage of pointers is particularly evident in large array traversal scenarios, being much faster than normal Python array access.

```python
import micropython

# Prepare a bytearray of length 10000 (large array)
ba = bytearray(10000)
for i in range(10000):
    ba[i] = i % 256

# Normal function: traverse bytearray, accumulate values
@timed_function('normal_bytearray_access')
def normal_bytearray_access(ba: bytearray) -> int:
    total = 0
    for i in range(10000):
        total += ba[i]
    return total

# Viper function: ptr8 pointer access, accumulate values (conversion at start)
@timed_function('viper_ptr8_access')
@micropython.viper
def viper_ptr8_access(ba) -> int:
    buf = ptr8(ba)
    total = 0
    for i in range(10000):
        total += buf[i]
    return total

# Call test (pointer access is much faster than normal access)
normal_bytearray_access(ba)
viper_ptr8_access(ba)
```

Terminal output:

```
>>> normal_bytearray_access(ba)
Function normal_bytearray_access Time = 71.562ms
1273080
>>> viper_ptr8_access(ba)
Function viper_ptr8_access Time =  3.142ms
1273080
```

> **Result Analysis**:
> - `normal_bytearray_access` took **71.562ms**
> - `viper_ptr8_access` took **3.142ms**
> - The Viper ptr8 version is **~23 times faster**!
> - The normal function's `ba[i]` goes through Python's bounds checking, object attribute lookup, etc. Viper's `buf[i]` directly calculates the memory address and accesses the byte, with no additional overhead.

**Compare the impact of pointer conversion location (inside loop vs. at function start)**:

```python
import micropython

# Prepare a bytearray of length 10000 (large array)
ba = bytearray(10000)
for i in range(10000):
    ba[i] = i % 256

# Viper function: repeated ptr8 conversion inside loop (inefficient)
@timed_function('viper_bad_convert')
@micropython.viper
def viper_bad_convert(ba) -> int:
    total = 0
    for i in range(10000):
        buf = ptr8(ba)    # Convert every loop, cumulative time
        total += buf[i]
    return total

# Viper function: single ptr8 conversion at start (efficient)
@timed_function('viper_good_convert')
@micropython.viper
def viper_good_convert(ba) -> int:
    buf = ptr8(ba)        # Convert only once
    total = 0
    for i in range(10000):
        total += buf[i]
    return total
    
# Call test
viper_bad_convert(ba)
viper_good_convert(ba)
```

Result:

```
>>> viper_bad_convert(ba)
Function viper_bad_convert Time = 16.371ms
1273080
>>> viper_good_convert(ba)
Function viper_good_convert Time =  3.130ms
1273080
```

> **Result Analysis**:
> - `viper_bad_convert` (conversion inside loop) took **16.371ms**
> - `viper_good_convert` (single conversion at start) took **3.130ms**
> - The difference is about **5 times**! In a loop of 10000 iterations, performing `ptr8(ba)` type conversion each time (each taking a few microseconds, accumulating significantly) is much slower than performing the conversion only once at the beginning, avoiding the repeated overhead.

Viper's integers are at the machine word level. On 32-bit hardware, arithmetic operations are performed modulo 2^32 (overflow truncation occurs). This is a trade-off of compatibility for performance, and this characteristic becomes more pronounced with large calculations.

### 2.2.4 Computation and Hardware Optimisation

#### 2.2.4.1 Replace Floating-Point Operations with Integer Operations

Chips without an FPU (Floating-Point Coprocessor) execute floating-point operations extremely slowly. Use integer operations in performance-critical sections, and only convert to floating-point in non-critical sections.

```python
from machine import ADC, Pin
import array

def timed_function(f, *args, **kwargs):
    myname = str(f).split(' ')[1]
    def new_func(*args, **kwargs):
        t = time.ticks_us()
        result = f(*args, **kwargs)
        delta = time.ticks_diff(time.ticks_us(), t)
        print('Function {} Time = {:6.3f}ms'.format(myname, delta/1000))
        return result
    return new_func

# 1. Pure integer operations: pre-allocated array + ADC reading (no floating-point)
# Pre-allocate an array of 100 int elements (contiguous memory, no dynamic allocation)
# Pure integer operations: read 16-bit integer ADC values
@timed_function
def adc_read_integer():
    adc_data = array.array('i', [0]*100)
    adc = ADC(Pin(26))
    
    for i in range(100):
        adc_data[i] = adc.read_u16()
    return adc_data

# 2. Includes floating-point operations: integer reading + floating-point conversion (voltage calculation)
# Step 1: Integer reading (same as above)
# Step 2: Floating-point conversion to voltage (reading/65535*3.3)
@timed_function
def adc_read_float():
    adc_data = array.array('i', [0]*100)
    adc = ADC(Pin(26))
  
    for i in range(100):
        adc_data[i] = adc.read_u16()
    
    voltage_data = [x/65535*3.3 for x in adc_data]
    return voltage_data
    
# Execute test, compare time
print("=== ADC Reading Performance Comparison ===")
integer_data = adc_read_integer()
float_data = adc_read_float()

# Print first 5 voltage values (verify functionality)
print("\nFirst 5 voltage values:", float_data[:5])
```

Terminal result:

```
=== ADC ===
>>> integer_data = adc_read_integer()
Function adc_read_integer Time =  2.428ms
>>> float_data = adc_read_float()
Function adc_read_float Time =  3.816ms
>>> print("\n5", float_data[:5])
5 [0.7333165, 0.7373449, 0.7397619, 0.7429847, 0.7462073]
```

> **Result Analysis**:
> - `adc_read_integer` (pure integer) took **2.428ms**
> - `adc_read_float` (with floating-point conversion) took **3.816ms**
> - The floating-point version is about **57%** slower than the integer version. This difference will be further amplified as the number of ADC samples increases.
> - **Optimisation Suggestion**: Perform only integer ADC readings inside the loop. Perform a single floating-point conversion after the loop (if needed), or only convert when displaying the output.

#### 2.2.4.2 Direct Register Read/Write

Bypass MicroPython's hardware abstraction layer and directly read/write chip registers, eliminating the overhead of method calls (e.g., fast LED blinking, high-frequency GPIO operations).

```python
from machine import Pin, mem32
import time
from micropython import const

def timed_function(f, *args, **kwargs):
    myname = str(f).split(' ')[1]
    def new_func(*args, **kwargs):
        t = time.ticks_us()
        result = f(*args, **kwargs)
        delta = time.ticks_diff(time.ticks_us(), t)
        print('Function {} Time = {:6.3f}ms'.format(myname, delta/1000))
        return result
    return new_func

# --------------------------
# Configure SIO registers (Raspberry Pi Pico specific)
# --------------------------
# SIO module base address (fixed for RP2040)
SIO_BASE = const(0xD0000000)
# Write all GPIO output values at once
GPIO_OUT     = SIO_BASE + 0x010   
# Atomic GPIO set
GPIO_OUT_SET = SIO_BASE + 0x014
# Atomic GPIO clear
GPIO_OUT_CLR = SIO_BASE + 0x018
# Atomic GPIO toggle
GPIO_OUT_XOR = SIO_BASE + 0x01C
# Atomic set GPIO to output mode
GPIO_OE_SET  = SIO_BASE + 0x024
# Bit mask for GPIO25 (onboard LED) (bit 25 corresponds to GPIO25)
PIN25_MASK = const(1 << 25)

# Initialisation: Set GPIO25 to output mode (execute only once, atomic operation) 
mem32[GPIO_OE_SET] = PIN25_MASK

# Initialise Pin object (execute only once)
led_pin = Pin(25, Pin.OUT)

# Method 1: Normal machine.Pin operation on GPIO25 (hardware abstraction layer, has overhead)
@timed_function
def led_pin_loop(loop_count):
    for _ in range(loop_count):
        led_pin.value(1)
        led_pin.value(0)

# Method 2: SIO register operation on GPIO25 (no abstraction layer, extremely efficient)
@timed_function
def led_sio_set_clr_loop(loop_count):
    set_reg = GPIO_OUT_SET   # Cache register address to local variable
    clr_reg = GPIO_OUT_CLR
    mask = PIN25_MASK
    for _ in range(loop_count):
        mem32[set_reg] = mask
        mem32[clr_reg] = mask

# Test execution speed (1000 toggles)
loop_count = 1000

led_pin_loop(loop_count)
led_sio_set_clr_loop(loop_count)
```

Result:

REPL code definition (showing SIO register address configuration):

```
>>> from machine import Pin, mem32
>>> import time
>>> from micropython import const
>>>
>>> SIO_BASE = const(0xD0000000)
>>> GPIO_OUT = SIO_BASE + 0x010
>>> GPIO_OUT_SET = SIO_BASE + 0x014
>>> GPIO_OUT_CLR = SIO_BASE + 0x018
>>> GPIO_OUT_XOR = SIO_BASE + 0x01C
>>> GPIO_OE_SET = SIO_BASE + 0x024
>>> PIN25_MASK = const(1 << 25)
>>> mem32[GPIO_OE_SET] = PIN25_MASK
>>> led_pin = Pin(25, Pin.OUT)
>>> @timed_function
... def led_pin_loop(loop_count):
...         for _ in range(loop_count):
...                 led_pin.value(1)
...                 led_pin.value(0)
```

Timing test results:

```
>>> loop_count = 1000
>>> led_pin_loop(loop_count)
Function led_pin_loop Time = 16.274ms
>>> led_sio_set_clr_loop(loop_count)
Function led_sio_set_clr_loop Time = 10.968ms
```

> **Result Analysis**:
> - `led_pin_loop` (machine.Pin method) took **16.274ms**
> - `led_sio_set_clr_loop` (SIO register method) took **10.968ms**
> - The register method is **~48% faster** (1.5x)
> - In the SIO loop function, caching `GPIO_OUT_SET`, `GPIO_OUT_CLR`, and `PIN25_MASK` into local variables reduces the overhead of global variable lookups, making the SIO performance advantage more prominent.
> - **Further Optimisation**: Combining the `@micropython.viper` decorator with `ptr32` pointers to directly write to registers can push GPIO toggle speeds into the MHz range.

#### 2.2.4.3 DMA Related Operations

In scenarios involving computation and hardware data interaction (e.g., batch ADC data acquisition, high-frequency GPIO signal output, sensor data stream reading), the CPU often spends a significant amount of time on **data transfer operations** (e.g., reading data from peripheral registers to memory, writing calculation results to GPIO registers), consuming resources needed for computation. The RP2040's DMA (Direct Memory Access) controller can autonomously perform batch data transfers between memory and peripherals/registers without CPU intervention. Its core optimisation value is: **freeing the CPU from tedious data transfer tasks, allowing it to focus on core computational logic**.

From a development and performance perspective, MicroPython's support for RP2040 DMA is relatively basic, only enabling simple batch data transfers. In contrast, C (Pico SDK) can fully configure DMA's transfer modes, trigger conditions, and data processing rules. Combined with direct register operations, it enables seamless optimisation of computation and hardware data interaction, making it the optimal choice for high-throughput, low-latency scenarios.

# 3. Optimisation Experiments

## 3.1 LCD Screen Optimisation

For this, please refer to:

[08 SPI Serial Peripheral Interface - Document Tutorial Draft](https://f1829ryac0m.feishu.cn/wiki/Qzagwgz3diFggSkR6rnc8oyunCe)

## 3.2 DMA Related Optimisation

For this, please refer to:

[19 DMA Direct Memory Access - Document Tutorial Draft](https://f1829ryac0m.feishu.cn/docx/OkDhdXUngoCUfKxDMcqc4BZJneh)

# 4. Optimisation Effect Summary

| Optimisation Method | Test Scenario | Time Before Optimisation | Time After Optimisation | Speedup Factor |
|---------------------|---------------|--------------------------|-------------------------|----------------|
| `@micropython.viper` (integer ops) | 1M `i*2+5` accumulations | 17221.732ms | 296.095ms | **~58x** |
| `viper ptr8` pointer access | 10k bytearray traversal | 71.562ms | 3.142ms | **~23x** |
| `ptr8` conversion moved outside loop | 10k pointer accesses | 16.371ms (bad) | 3.130ms (good) | **~5x** |
| `memoryview` instead of slice | Passing bytearray slice | 0.095ms | 0.079ms | ~20% |
| Cache object reference | 100 attribute access writes | 0.999ms | 1.039ms (example) | ~same (more effective in large loops) |
| `const()` constant | 100k loop accessing constant | 1120.892ms | 1120.690ms | ~same in REPL (more effective with precompilation) |
| Integer instead of float (ADC) | 100 ADC samples | 3.816ms (with float conversion) | 2.428ms (pure integer) | **~57%** |
| SIO register instead of `machine.Pin` | 1000 GPIO toggles | 16.274ms | 10.968ms | **~48%** |

# References

- [MicroPython Official Performance Optimisation Documentation](https://docs.micropython.org/en/latest/reference/speed_python.html)
- [MicroPython viper Code Emitter Description](https://docs.micropython.org/en/latest/reference/speed_python.html#the-viper-code-emitter)
- [RP2040 Datasheet - SIO Module](https://datasheets.raspberrypi.com/rp2040/rp2040-datasheet.pdf)
