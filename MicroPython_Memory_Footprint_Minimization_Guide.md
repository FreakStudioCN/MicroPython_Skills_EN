# How to Minimize MicroPython Memory Usage

# 1. Prerequisites

MicroPython's core application scenario is **microcontrollers (such as Raspberry Pi Pico, ESP32, ESP8266)**, whose hardware resources are orders of magnitude different from computers/servers — typically RAM (runtime memory) is only tens of KB to hundreds of KB, and Flash (storage memory) is only a few MB. Python code itself is dynamic and consumes certain memory resources at runtime. Therefore, when using MicroPython on microcontrollers, memory optimization is key to ensuring stable program operation. This article will detail MicroPython memory optimization methods, annotating and explaining specialized terms that may be difficult for beginners to understand, while providing test code that can be run in the REPL (MicroPython interactive command line) for core knowledge points, helping you learn and verify simultaneously.

Before learning specific optimization methods, understanding the following core specialized terms will make subsequent content easier to grasp:

---

# 2. Optimization Methods

## 2.1 Basic Concepts

Before learning specific optimization methods, understanding the following core specialized terms will make subsequent content easier to grasp:

- **RAM (Random Access Memory)**: Also known as runtime memory, it is the area in the microcontroller used for temporarily storing running code and data. It is characterized by fast read/write speeds, but data is lost when power is off, and its capacity is extremely small (Raspberry Pi Pico has 264KB of RAM).
- **Flash**: Also known as storage memory, it is the area in the microcontroller used for permanently storing firmware, user code, and data. It is characterized by data retention when power is off, has larger capacity than RAM, but slower read/write speeds (Raspberry Pi Pico has 2MB of Flash).
- **Bytecode**: MicroPython does not directly execute human-written source code (.py files). Instead, it first compiles the source code into an intermediate code between source code and machine code (i.e., bytecode), which is then executed by the MicroPython interpreter. Bytecode is smaller in size and more efficient to execute.
- **Firmware**: The underlying software burned into the microcontroller's Flash, containing core functions such as the MicroPython interpreter and hardware drivers. It is the foundation for MicroPython operation.
- **REPL**: Read-Eval-Print Loop, which is MicroPython's interactive command-line tool (such as Thonny's Shell panel, serial tool's interactive window). It allows you to input code line by line and execute it immediately, making it an essential tool for beginners to debug and test code.
- **SPI Bus**: A Serial Peripheral Interface, a common protocol for communication between microcontrollers and external devices (such as SD cards, sensors), characterized by simple wiring and fast transmission speed.

---

## 2.2 Installing an SD Card

Development boards that support MicroPython can expand memory by inserting an SD card. First, the card needs to be formatted as **FAT/FAT32 format (a common file system format supported by most devices, and the necessary format for MicroPython to recognize the SD card)**. Usually, the SD card is mounted via the SPI bus. After inserting the card, MicroPython will boot from the SD card. If there are `boot.py` and `main.py` on the SD card, they will also be automatically executed at startup.

We can also boot from internal flash while using the SD card to save data. In this case, you need to create a `SKIPSD` file in the root directory of the SD card. When the system starts and detects this file on the SD card, it will ignore the SD card and still boot from internal Flash.

For more information on using SD cards with MicroPython, see the following tutorial:

[08 SPI Serial Peripheral Interface - Document Tutorial Draft](https://f1829ryac0m.feishu.cn/docx/G4P7dNfYso36YLxVlCQc6V4snUc)

For using SD cards on the Raspberry Pi Pico, refer to the tutorial: (To be added)

---

## 2.3 Using Frozen Modules and Frozen Bytecode

In actual development, we often have multiple .py files storing different code. The `main.py` file runs the main business process, and `main.py` references other .py files/modules/packages through `import` statements to complete the entire work (e.g., classes for other sensors). The `main.py` file and other files are often located in the root directory of the development board's file system.

Loading modules and packages from Python files on the file system to store and run code has some significant limitations. This Python code must be loaded and processed by the MicroPython interpreter, a process that consumes time and memory. In some cases, these code files are too large to be loaded into Flash memory and processed by the MicroPython interpreter. Frozen modules and frozen bytecode can compile Python code into native code/bytecode and store it with the firmware, thereby compressing memory. Once the code is frozen, MicroPython can quickly load and interpret it without requiring much memory and processing time.

### 2.3.1 Python Bytecode

MicroPython code is first compiled into bytecode, and then the interpreter executes the bytecode. MicroPython's bytecode is an intermediate language similar to assembly instructions. One MicroPython statement corresponds to several bytecode instructions. The interpreter executes the bytecode instructions sequentially, thereby completing program execution.

The code is pre-compiled into bytecode, avoiding the need to compile MicroPython source code at load time. Bytecode can be executed directly from Flash without needing to be copied into RAM. Similarly, any constant objects (such as strings, tuples, etc.) are also loaded from ROM. This allows more memory to be available for the application. On devices without a file system, this is the only way to load Python code.

### 2.3.2 Steps to Generate Frozen Modules and Bytecode

The steps to generate frozen modules and bytecode are typically as follows:

1. **Prepare MicroPython Source Code**: Clone the MicroPython repository from GitHub, ensuring a complete compilation environment locally (e.g., `arm-none-eabi-gcc`).
2. **Place Modules in `ports/<platform>/modules/` Directory**: For example, `ports/rp2/modules/`, place the `.py` files to be frozen in this directory.
3. **Compile Firmware**: Execute the `make` command to recompile the firmware. The compilation process will automatically compile the `.py` files in this directory into frozen bytecode and embed them into the firmware.
4. **Burn Firmware to the Development Board**: Burn the generated `.uf2` or `.bin` firmware to the device.
5. **Normal `import` in Code**: The frozen module can be loaded using the `import` statement like a regular module, no additional operations required.

### 2.3.3 Main Features of MicroPython's Frozen Modules

The main features of MicroPython's frozen modules are:

- **Storage Location**: Frozen modules are stored in Flash (ROM) rather than RAM, not occupying precious runtime memory.
- **Fast Loading Speed**: No need for runtime compilation; executed directly by the interpreter, resulting in faster startup.
- **RAM Saving**: Bytecode and constant objects run directly from Flash, eliminating the need to copy them into RAM.
- **Suitable for Devices Without File System**: On devices without a file system, this is the only way to load Python code.
- **Determined at Compile Time**: Frozen modules are determined when the firmware is compiled and cannot be dynamically modified at runtime.

For tutorials on using frozen modules and frozen bytecode, refer to: (To be added)

---

## 2.4 Using .mpy Files

Pre-compile Python modules into bytecode (also known as .mpy files (MicroPython's pre-compiled bytecode file format, different from standard Python's .pyc files)), and then copy them to the development board. **The advantage of this is that it skips the pre-compilation phase on the development board, thereby avoiding the lack of RAM resources during this process. Unfortunately, this method still requires the development board to load the module into RAM.**

Convert the Python module into a .c file, which is compiled into the firmware itself. This has the advantages of the above method + the advantage of running the module from flash instead of loading it into RAM.

For steps on generating and using .mpy files, refer to: [How to Speed Up MicroPython Execution](https://f1829ryac0m.feishu.cn/docx/BVTZdXCMCobRbFxuM8jcNCEMndc)

---

## 2.5 Using const Constants

### 2.5.1 MicroPython Namespace and Scope

All modules loaded into RAM in MicroPython are placed in `sys.modules`. `sys.modules` is a global dictionary that is loaded into memory from the start of the Python program. It is used to store the names and objects of all currently imported (loaded) modules. In MicroPython's module lookup, it acts as a cache, avoiding repeated loading of modules.

When a program imports a module, it first checks whether `sys.modules` contains this module name. If it exists, it only needs to add the module's name to the current module's `Local` namespace. If it does not exist, it needs to search for the module file in the `sys.path` directories according to the module name. After finding it, the module is loaded into memory and added to the `sys.modules` dictionary. Finally, the module's name is added to the current module's `Local` namespace.

Next, let's test this in the terminal's `REPL`:

```python
import sys

# sys.modules shows all currently imported (loaded) module names and module objects
print("Initial modules:")
print(list(sys.modules.keys()))

# Define a global variable
global_var = "I am a global variable"

# enclosing_var is a closure outer variable
# local_var is a local variable inside the inner_function
# len is a built-in function
# Attempt to modify the global variable global_var (not using global creates a local variable)
# The closure outer layer cannot access the inner local variable
def scope_demo():
    enclosing_var = "I am an enclosing variable (outer of closure)"
    
    def inner_function():
        local_var = "I am a local variable"
        
        print("Local variable:", local_var)
        print("Enclosing variable:", enclosing_var)
        print("Global variable:", global_var)
        print("Built-in function example:", len("test"))
        
        try:
            global_var = "Trying to modify"
        except:
            pass
    
    inner_function()
    
    try:
        print(local_var)
    except:
        print("Cannot access local variable")

# Use global to modify the global variable
def modify_global():
    global global_var
    global_var = "Modified global variable"
    print("Inside function:", global_var)

# Using nonlocal in multi-level nesting
# Modify the variable outer_var in the enclosing scope from the inner middle() and inner() functions
def nonlocal_demo():
    outer_var = "Outer layer"
    
    def middle():
        nonlocal outer_var
        outer_var = "Modified by middle layer"
        
        def inner():
            nonlocal outer_var
            outer_var = "Modified by inner layer"
        
        inner()
        print("In middle:", outer_var)
    
    middle()
    print("In nonlocal_demo:", outer_var)

# Scope demonstration
print("\n=== Scope Demonstration ===")
scope_demo()
print("Access in global scope:", global_var)

# Modify global variable
print("\n=== Modify Global Variable ===")
modify_global()
print("In global scope:", global_var)

# nonlocal demonstration
print("\n=== nonlocal Demonstration ===")
nonlocal_demo()

# Attempt to access built-in scope
import builtins
print("Built-in module:", builtins)
```

Terminal output is as follows:

**Figure 1: REPL input for scope_demo function definition process**

```
(base) PS G:\test_microMLP> mpremote
Connected to MicroPython at COM65
Use Ctrl-] or Ctrl-x to exit this shell

>>> import sys
>>> print("Initial modules:")
Initial modules:
>>> print(list(sys.modules.keys()))
['rp2']
>>> global_var = "I am a global variable"
>>> def scope_demo():
...     enclosing_var = "I am an enclosing variable (outer of closure)"
...
...     def inner_function():
...         local_var = "I am a local variable"
...
...         print("Local variable:", local_var)
...         print("Enclosing variable:", enclosing_var)
...         print("Global variable:", global_var)
...         print("Built-in function example:", len("test"))
...
...         try:
...             global_var = "Trying to modify"
...         except:
...             pass
...
...     inner_function()
...
...     try:
...         print(local_var)
...     except:
...         print("Cannot access local variable")
...
>>> scope_demo()
```

**Figure 2: scope_demo execution result (local variable access exception)**

```
>>> scope_demo()
Local variable: I am a local variable
Enclosing variable: I am an enclosing variable (outer of closure)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 17, in scope_demo
  File "<stdin>", line 9, in inner_function
NameError: local variable referenced before assignment
```

> **Note**: Inside `inner_function`, when the assignment statement `global_var = "Trying to modify"` appears in the `try` block, the Python compiler considers `global_var` to be a local variable. Therefore, it throws `NameError: local variable referenced before assignment` at `print("Global variable:", global_var)` because the local variable is referenced before assignment.

**Figure 3: modify_global function definition and execution**

```
>>> def modify_global():
...     global global_var
...     global_var = "Modified global variable"
...     print("Inside function:", global_var)
...
>>> modify_global()
```

**Figure 4: nonlocal_demo execution result**

```
>>> def nonlocal_demo():
...     outer_var = "Outer layer"
...
...     def middle():
...         nonlocal outer_var
...         outer_var = "Modified by middle layer"
...
...         def inner():
...             nonlocal outer_var
...             outer_var = "Modified by inner layer"
...
...         inner()
...         print("In middle:", outer_var)
...
...     middle()
...     print("In nonlocal_demo:", outer_var)
...
>>> nonlocal_demo()
In middle: Modified by inner layer
In nonlocal_demo: Modified by inner layer
```

**Figure 5: Built-in module access**

```
>>> import builtins
>>> print("Built-in module:", builtins)
Built-in module: <module 'builtins'>
>>>
```

> **LEGB Rule Summary**: The variable lookup order in MicroPython is Local → Enclosing → Global → Built-in, i.e., the LEGB rule. The `global` keyword is used to declare access to a global variable within a function, and the `nonlocal` keyword is used to declare access to an outer (non-global) variable in a nested function.

---

### 2.5.2 Definition and Optimization Principle of const Constants

Constants are values that do not change during program execution. They are generally used to store immutable data, such as mathematical constants, configuration information, etc. In microcontrollers, constants are stored in ROM/Flash memory, which can save RAM space.

The `const` keyword in MicroPython is used to declare that an expression is a constant so that the compiler can optimize it. In the two cases where a constant is assigned to a variable, the compiler will avoid writing a lookup for the constant name by substituting the literal value of the constant. This saves bytecode, thereby saving RAM.

**However, it is important to note that when using the `const` keyword, you need to compile it into an .mpy file using the `mpy-cross` tool for it to take effect.**

```python
from micropython import const
# Garbage collection module, used to view memory usage
import gc

# Step 1: View initial memory usage
# Force garbage collection to free unused memory
gc.collect()
initial_free = gc.mem_free()
print(f"Initial free RAM: {initial_free} bytes")

# Step 2: Define const constants
# Public constants (accessible outside the module, only occupy minimal RAM)
CONST_X = const(123)
CONST_Y = const(2 * 123 + 1)
# Private constants (start with underscore, not accessible outside the module, occupy no RAM)
_COLS = const(0x10)
ROWS = const(33)

# Step 3: Use constants
a = ROWS
b = _COLS
print(f"CONST_X: {CONST_X}, CONST_Y: {CONST_Y}")
print(f"a: {a}, b: {b}")

# Step 4: View memory usage after defining constants
gc.collect()
after_free = gc.mem_free()
print(f"Free RAM after defining constants: {after_free} bytes")
print(f"RAM change: {after_free - initial_free} bytes (positive means memory freed)")

# Supplement: Compare the difference between normal variables and const constants
# Normal variable, stored in RAM
normal_var = 123  
gc.collect()
after_normal = gc.mem_free()
print(f"Free RAM after defining normal variable: {after_normal} bytes")
print(f"Normal variable uses {after_free - after_normal} more bytes than const")
```

The `ROWS` value will occupy at least two machine words, one each for the key and value in the global dictionary. The existence in the dictionary is necessary because another module might import or use it. This RAM can be saved by prefixing the name with an underscore, as shown with `_COLS`: this symbol is not visible outside the module, so it does not occupy RAM.

Terminal output is as follows:

**Figure 6: const constant test output**

```
Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

Try the new cross-platform PowerShell https://aka.ms/pscore6

Loading personal and system profiles took 570 milliseconds.
(base) PS G:\test_microMLP> mpremote
Connected to MicroPython at COM65
Use Ctrl-] or Ctrl-x to exit this shell

>>> from micropython import const
>>> import gc
>>> gc.collect()
>>> initial_free = gc.mem_free()
>>> print(f"Initial free RAM: {initial_free} bytes")
Initial free RAM: 228416 bytes
>>> CONST_X = const(123)
>>> CONST_Y = const(2 * 123 + 1)
>>> _COLS = const(0x10)
>>> ROWS = const(33)
>>> a = ROWS
>>> b = _COLS
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
NameError: name '_COLS' isn't defined
>>> print(f"CONST_X: {CONST_X}, CONST_Y: {CONST_Y}")
CONST_X: 123, CONST_Y: 247
>>> print(f"a: {a}, b: {b}")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
NameError: name 'b' isn't defined
>>> gc.collect()
>>> after_free = gc.mem_free()
>>> print(f"Free RAM after defining constants: {after_free} bytes")
Free RAM after defining constants: 228176 bytes
>>> print(f"RAM change: {after_free - initial_free} bytes (positive means memory freed)")
RAM change: -240 bytes (positive means memory freed)
>>> normal_var = 123
>>> gc.collect()
>>> after_normal = gc.mem_free()
>>> print(f"Free RAM after defining normal variable: {after_normal} bytes")
Free RAM after defining normal variable: 228000 bytes
>>> print(f"Normal variable uses {after_free - after_normal} more bytes than const")
Normal variable uses 176 more bytes than const
>>>
```

> **Key Observations**:
> - `_COLS` starts with an underscore, making it a private constant. Directly accessing `b = _COLS` in the REPL throws a `NameError` because it does not appear in the global namespace.
> - The normal variable `normal_var = 123` occupies 176 more bytes of RAM than the `const` constant, verifying the memory optimization effect of `const`.
> - `CONST_Y = const(2 * 123 + 1)` correctly outputs 247, indicating that `const()` supports compile-time integer expressions.

The argument to `const()` must be any value that evaluates to an integer at compile time.

```python
from micropython import const

ROWS = const(33)
# The following usage throws an error
COLS = const(0x10+ROWS)
# Correct usage
COLS = const(0x10+33)
```

Terminal output is as follows:

**Figure 7: Example of incorrect const usage**

```
Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

Try the new cross-platform PowerShell https://aka.ms/pscore6

Loading personal and system profiles took 808 milliseconds.
(base) PS G:\test_microMLP> mpremote
Connected to MicroPython at COM65
Use Ctrl-] or Ctrl-x to exit this shell

>>> from micropython import const
>>> ROWS = const(33)
>>> COLS = const(0x10+ROWS)
Traceback (most recent call last):
  File "<stdin>", line 1
SyntaxError: not a constant
>>> COLS = const(0x10+33)
>>>
```

> `const(0x10+ROWS)` throws `SyntaxError: not a constant` because `ROWS` is a symbol name that cannot be evaluated at compile time in the current REPL context. The correct approach is to write the literal value directly: `const(0x10+33)`.

---

## 2.6 Reducing Unnecessary Object Creation

In MicroPython, **the creation of objects (such as strings, lists, dictionaries, bytearrays, numeric containers, etc.) consumes RAM resources. Due to the extremely small RAM capacity of microcontrollers, frequently creating and destroying objects not only directly consumes memory but also leads to memory fragmentation** (i.e., many small free memory blocks appear in RAM, which cannot be used by large objects), ultimately causing out-of-memory issues.

It is important to note that the `sys.getsizeof()` function from standard Python is not supported on most MicroPython platforms (this is a MicroPython feature to keep the firmware lean). Therefore, we can use the `struct` module to pack variables/data into a byte stream, then use the `len()` function to get the length of the byte stream, thereby testing the byte size occupied by the data. At the same time, we can use the `gc` module (garbage collection) to view the actual changes in RAM usage, completing the comparison before and after optimization.

### 2.6.1 String Operation Optimization (Avoid Temporary String Objects)

Strings in MicroPython are immutable objects. When using `+` to concatenate strings, each `+` creates a temporary string object (e.g., `"a"+"b"+"c"` first creates `"ab"`, then creates `"abc"`, totaling 2 temporary objects). These temporary objects consume additional RAM and are frequently cleaned up by the garbage collector, affecting performance.

Optimization idea:

```python
# Static string concatenation (merged at compile time, no temporary objects)
static_str = "MicroPython" "Memory" "Opt"
print("Static string:", static_str)

# Dynamic string (recommend format, reduces temporary objects)
temp = 25.5
press = 101325
dynamic_str = "Temp: {:.2f}, Press: {:d}".format(temp, press)
print("Dynamic string:", dynamic_str)
```

### 2.6.2 Buffer Reuse in Hardware Operations (UART/SPI/I2C Scenarios)

In MicroPython hardware operations (such as SPI reading sensor data, UART receiving data), frequently creating new byte buffers (`bytearray`/`bytes`) consumes a lot of RAM. For example, creating `buf = bytearray(10)` each time you read a sensor, and looping 100 times, creates 100 buffer objects.

We can pre-allocate a buffer and reuse it in loops/multiple operations, creating the object only once, completely avoiding the creation of temporary buffers.

Below, we pre-allocate a UART receive buffer and reuse it in a loop, avoiding repeated creation of `bytearray`.

```python
from machine import UART, Pin

# Initialize UART
uart = UART(0, baudrate=9600, tx=Pin(0), rx=Pin(1))
# Pre-allocate buffer (created only once)
uart_buf = bytearray(16)

# Loop receive (reuse buffer)
for _ in range(5):
    uart.readinto(uart_buf)
    print("UART data:", uart_buf)
```

Below, we pre-allocate an SPI read/write buffer and reuse it to reduce memory overhead.

```python
from machine import SPI, Pin

# Initialize SPI
spi = SPI(0, baudrate=1000000, sck=Pin(2), mosi=Pin(3), miso=Pin(4))
# Pre-allocate buffer
spi_buf = bytearray(8)

# Loop read/write (reuse buffer)
for _ in range(5):
    spi.readinto(spi_buf)
    print("SPI data:", spi_buf)
```

### 2.6.3 Numeric Storage Optimization (Use struct/bytearray Instead of Lists)

In MicroPython, storing numeric values in a list (e.g., `[1, 2, 3, 255]`) consumes more RAM because each integer object in the list has additional memory overhead (e.g., an `int` type in MicroPython occupies 4 bytes, and the list itself has pointer overhead).

Optimization idea:

```python
# 1. str vs bytes (memory usage is the same for ASCII scenarios)
# String (ASCII)
s = 'the quick brown fox'  
# Bytes (1 byte per char)
b = b'the quick brown fox' 
print("String:", s)
print("Bytes:", b)

# 2. String to bytes conversion (note: conversion creates new objects)
# str -> bytes
s_to_b = s.encode()  
# bytes -> str
b_to_s = b.decode()  
print("Str->Bytes:", s_to_b)
print("Bytes->Str:", b_to_s)

# 3. bytes supports string methods (e.g., lstrip)
foo = b'  empty whitespace'
foo_stripped = foo.lstrip()
print("Stripped bytes:", foo_stripped)
```

Terminal output is as follows:

**Figure 8: bytes and str operation test output**

```
Copyright (C) Microsoft Corporation. All rights reserved.

Try the new cross-platform PowerShell https://aka.ms/pscore6

Loading personal and system profiles took 685 milliseconds.
(base) PS G:\test_microMLP> mpremote
Connected to MicroPython at COM65
Use Ctrl-] or Ctrl-x to exit this shell

>>> s = 'the quick brown fox'
>>> b = b'the quick brown fox'
>>> print("String:", s)
String: the quick brown fox
>>> print("Bytes:", b)
Bytes: b'the quick brown fox'
>>> s_to_b = s.encode()
>>> b_to_s = b.decode()
>>> print("Str->Bytes:", s_to_b)
Str->Bytes: b'the quick brown fox'
>>> print("Bytes->Str:", b_to_s)
Bytes->Str: the quick brown fox
>>> foo = b'  empty whitespace'
>>> foo_stripped = foo.lstrip()
>>> print("Stripped bytes:", foo_stripped)
Stripped bytes: b'empty whitespace'
>>>
```

### 2.6.4 Avoid Creating Temporary Containers in Loops

Frequently creating temporary containers in loops (e.g., `for _ in range(100): temp = [1,2,3]`) creates a large number of temporary objects, consuming RAM. The optimization idea is to move the temporary container outside the loop, reuse the object, or use generators/iterators instead of temporary lists.

```python
import gc
import struct

# Initialize garbage collection, establish test baseline
gc.collect()
ram_init = gc.mem_free()

# Define the number of loop iterations (simulate multiple iteration scenarios)
loop_times = 100

# ---------------------- Inefficient Implementation: Create Temporary List Inside Loop ----------------------
# Create a new list container each time inside the loop, generating many temporary objects
gc.collect()
ram_slow_before = gc.mem_free()
total_slow = 0

# Create a new list [1,2,3,4,5] each loop
# struct tests the byte size of the list data (packed as 5 integers, using 'i' format)
for _ in range(loop_times):
    temp_list = [1, 2, 3, 4, 5]
    total_slow += sum(temp_list)
    
    size_slow = len(struct.pack("5i", *temp_list))
# Record RAM after inefficient implementation
ram_slow_after = gc.mem_free()

# ---------------------- Efficient Implementation 1: Create List Outside Loop, Reuse Container ----------------------
# List created only once, reused inside the loop
gc.collect()
ram_fast1_before = gc.mem_free()
total_fast1 = 0

# Temporary list moved outside the loop (created only once)
# struct tests the byte size of the reused list (consistent with inefficient implementation)
temp_list_reuse = [1, 2, 3, 4, 5]
for _ in range(loop_times):
    total_fast1 += sum(temp_list_reuse)
    
    size_fast1 = len(struct.pack("5i", *temp_list_reuse))
# Record RAM after efficient implementation 1
ram_fast1_after = gc.mem_free()

# ---------------------- Efficient Implementation 2: Use Generator, Replace Temporary List ----------------------
# Define generator function (created only once, iterator reused inside loop)
# Yield elements directly, avoiding tuple creation
def num_generator():
    yield 1
    yield 2
    yield 3
    yield 4
    yield 5
    
# Generator does not create a complete list, only generates elements iteratively, lowest memory overhead
gc.collect()
ram_fast2_before = gc.mem_free()
total_fast2 = 0

# Generator expression (no temporary list object)
# struct tests the byte size of the generator data (packed as 5 integers)
for _ in range(loop_times):
    total_fast2 += sum(num_generator())
    size_fast2 = len(struct.pack("5i", 1, 2, 3, 4, 5))
# Record RAM after efficient implementation 2
ram_fast2_after = gc.mem_free()

# Output test results (no Chinese)
print("Slow total:", total_slow)
print("Slow RAM change:", ram_slow_before - ram_slow_after)
print("Slow data size:", size_slow)

print("Fast1 total:", total_fast1)
print("Fast1 RAM change:", ram_fast1_before - ram_fast1_after)
print("Fast1 data size:", size_fast1)

print("Fast2 total:", total_fast2)
print("Fast2 RAM change:", ram_fast2_before - ram_fast2_after)
print("Fast2 data size:", size_fast2)
```

Terminal output is as follows:

**Figure 9: Loop temporary container test output**

```
...     yield 2
...     yield 3
...     yield 4
...     yield 5
...
>>> gc.collect()
>>> ram_fast2_before = gc.mem_free()
>>> total_fast2 = 0
>>> for _ in range(loop_times):
...     total_fast2 += sum(num_generator())
...     size_fast2 = len(struct.pack("5i", 1, 2, 3, 4, 5))
...
>>> ram_fast2_after = gc.mem_free()
>>> print("Slow total:", total_slow)
Slow total: 1500
>>> print("Slow RAM change:", ram_slow_before - ram_slow_after)
Slow RAM change: 10528
>>> print("Slow data size:", size_slow)
Slow data size: 20
>>>
>>> print("Fast1 total:", total_fast1)
Fast1 total: 1500
>>> print("Fast1 RAM change:", ram_fast1_before - ram_fast1_after)
Fast1 RAM change: 5888
>>> print("Fast1 data size:", size_fast1)
Fast1 data size: 20
>>>
>>> print("Fast2 total:", total_fast2)
Fast2 total: 1500
>>> print("Fast2 RAM change:", ram_fast2_before - ram_fast2_after)
Fast2 RAM change: 8720
>>> print("Fast2 data size:", size_fast2)
Fast2 data size: 20
>>>
```

> **Result Analysis**:
>
> | Implementation Method | RAM Change (bytes) | Description |
> |---|---|---|
> | Create temporary list inside loop (inefficient) | 10528 | 100 loops create 100 list objects, highest RAM consumption |
> | Reuse list outside loop (efficient 1) | 5888 | List created only once, minimal memory overhead |
> | Use generator (efficient 2) | 8720 | Generator object created each call, overhead between the two |
>
> In the scenario of repeating 5 fixed numbers 100 times, list reuse is indeed more memory-efficient than generators; when dealing with large amounts of data or streaming data, generators avoid loading all data at once.

### 2.6.5 Avoid Runtime Compiler Execution (`eval`/`exec`)

`eval()`/`exec()` calls the MicroPython compiler at runtime, creating many temporary objects and consuming a lot of RAM. Replacing them with direct calculation or `json` serialization can reduce memory overhead.

```python
import gc
import struct
import json

# ---------------------- Inefficient: Using eval ----------------------
gc.collect()
ram_eval_before = gc.mem_free()
# Avoid using eval, here only for comparison
res_eval = eval("1 + 2 * 3 + 4 / 2")
ram_eval_after = gc.mem_free()

# ---------------------- Efficient: Direct Calculation ----------------------
gc.collect()
ram_calc_before = gc.mem_free()
res_calc = 1 + 2 * 3 + 4 / 2
ram_calc_after = gc.mem_free()

# ---------------------- Efficient: ujson Serialization ----------------------
gc.collect()
ram_json_before = gc.mem_free()
# Serialization
data = {"temp": 25.5, "press": 101325}
json_data = json.dumps(data)
# Deserialization
size_json = len(struct.pack(f"{len(json_data.encode())}s", json_data.encode()))
data_loaded = json.loads(json_data)
ram_json_after = gc.mem_free()

# Output test results
print("Eval RAM change:", ram_eval_before - ram_eval_after)
print("Calc RAM change:", ram_calc_before - ram_calc_after)
print("JSON size:", size_json)
print("JSON RAM change:", ram_json_before - ram_json_after)
```

Terminal output is as follows:

**Figure 10: eval vs direct calculation vs json memory comparison**

```
>>> import gc
>>> import struct
>>> import json
>>> gc.collect()
>>> ram_eval_before = gc.mem_free()
>>> res_eval = eval("1 + 2 * 3 + 4 / 2")
>>> ram_eval_after = gc.mem_free()
>>> gc.collect()
>>> ram_calc_before = gc.mem_free()
>>> res_calc = 1 + 2 * 3 + 4 / 2
>>> ram_calc_after = gc.mem_free()
>>> gc.collect()
>>> ram_json_before = gc.mem_free()
>>> data = {"temp": 25.5, "press": 101325}
>>> json_data = json.dumps(data)
>>> size_json = len(struct.pack(f"{len(json_data.encode())}s", json_data.encode()))
>>> data_loaded = json.loads(json_data)
>>> ram_json_after = gc.mem_free()
>>> print("Eval RAM change:", ram_eval_before - ram_eval_after)
Eval RAM change: 592
>>> print("Calc RAM change:", ram_calc_before - ram_calc_after)
Calc RAM change: 384
>>> print("JSON size:", size_json)
JSON size: 31
>>> print("JSON RAM change:", ram_json_before - ram_json_after)
JSON RAM change: 1488
>>>
```

> **Result Analysis**:
>
> | Operation Method | RAM Change (bytes) | Description |
> |---|---|---|
> | `eval()` dynamic parsing | 592 | Calls the compiler, generates many temporary objects |
> | Direct calculation | 384 | No compilation overhead, lowest memory consumption |
> | `json.dumps()/loads()` | 1488 | Serializes strings and dictionaries, generates more temporary objects |
>
> We can see that dynamic parsing (`eval()`) and JSON processing create many hidden temporary objects. This is a typical manifestation of memory fragmentation on microcontrollers — seemingly simple operations hide exponential memory consumption.

### 2.6.6 Storing Strings in Flash (qstr Mechanism)

MicroPython's `qstr` (quantified string) mechanism stores repeated strings in Flash instead of RAM, reducing RAM usage. We can use `micropython.qstr_info()` to view the string storage status, `struct` to test the byte size of strings, and `gc` to view RAM changes.

```python
import gc
import struct
import micropython

# Initialize garbage collection
gc.collect()
ram_init = gc.mem_free()

# Define test strings (will be processed by the qstr mechanism)
s1 = "MicroPython"
# Repeated string, reuses the qstr in Flash
s2 = "MicroPython"  

# struct tests the byte size of the string
size_s = len(struct.pack(f"{len(s1.encode())}s", s1.encode()))

# Print qstr information (1 means detailed output)
micropython.qstr_info(1)

# Record RAM
ram_after = gc.mem_free()

# Output test results
print("String size:", size_s)
print("RAM free:", ram_after)
```

Terminal output is as follows:

**Figure 11: qstr string storage mechanism test**

```
Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

Try the new cross-platform PowerShell https://aka.ms/pscore6

Loading personal and system profiles took 675 milliseconds.
(base) PS G:\test_microMLP> mpremote
Connected to MicroPython at COM65
Use Ctrl-] or Ctrl-x to exit this shell

>>> import gc
>>> import struct
>>> import micropython
>>> gc.collect()
>>> ram_init = gc.mem_free()
>>> s1 = "MicroPython"
>>> s2 = "MicroPython"
>>> size_s = len(struct.pack(f"{len(s1.encode())}s", s1.encode()))
>>> micropython.qstr_info(1)
qstr pool: n_pool=1, n_qstr=5, n_str_data_bytes=26, n_total_bytes=170
Q(ram_init)
Q(s1)
Q(s2)
Q(size_s)
Q({}s)
>>> ram_after = gc.mem_free()
>>> print("String size:", size_s)
String size: 11
>>> print("RAM free:", ram_after)
RAM free: 227152
>>>
```

We can see that the two strings `s1` and `s2` have the same content, both being `"MicroPython"`. MicroPython's qstr mechanism reuses the same string. In fact, `s1` and `s2` point to the same string object in memory, saving the memory required for storing the same string repeatedly.

`qstr` information analysis is as follows:

```
>>> micropython.qstr_info(1)
qstr pool: n_pool=1, n_qstr=5, n_str_data_bytes=26, n_total_bytes=170
Q(ram_init)
Q(s1)
Q(s2)
Q(size_s)
Q({}s)
```

Key information parsing:

| Field | Value | Meaning |
|---|---|---|
| `n_pool` | 1 | Number of qstr pools, currently 1 qstr pool |
| `n_qstr` | 5 | Number of qstrs currently in RAM (the variable names themselves, not the string content) |
| `n_str_data_bytes` | 26 | Total bytes of variable name string data (sum of bytes for `ram_init`+`s1`+`s2`+`size_s`+`{}s`) |
| `n_total_bytes` | 170 | Total bytes occupied by the qstr pool (including metadata, alignment, etc.) |

> **Note**: The string `"MicroPython"` is not listed in the qstrs because MicroPython stores it as read-only data in Flash, not in RAM. The listed `Q(s1)` and `Q(s2)` are the variable names `s1` and `s2` themselves being interned as qstrs, not the string value `"MicroPython"`.

---

## 2.7 Heap and Garbage Collection Mechanism

The heap is a dynamically allocated memory region used to store object instances and dynamic data structures created during program runtime (such as lists, dictionaries, instances of custom classes in Python):

When you execute `lst = [1,2,3]`, Python will:

1. Allocate a memory region in the **heap** to store the list object `[1,2,3]`
2. Create the variable `lst` on the **stack**, which serves as a reference (pointer) to that heap memory region

If an object in the heap has no references pointing to it (i.e., the program can no longer access the object), that object becomes "garbage": for example, after executing `lst = None`, the `[1,2,3]` object originally in the heap loses its reference and becomes garbage.

In MicroPython, the GC (Garbage Collection) can automatically identify and clean up garbage objects in the heap, freeing occupied memory and preventing memory leaks (where memory is continuously occupied by useless objects, causing the program's available memory to decrease and eventually crash).

Garbage collection can be triggered in the following two ways:

1. **Automatic Trigger**: Automatically detected and triggered by the MicroPython runtime when allocating memory (enabled via `gc.enable()`, enabled by default). You can set a cumulative allocation threshold using `gc.threshold(n)`. When the cumulative allocation reaches the threshold, a GC is automatically triggered.
2. **Manual Trigger**: Manually trigger garbage collection by calling `gc.collect()`, suitable for actively freeing memory before performing large memory operations.

The core methods of MicroPython's `gc` module are as follows:

| Method | Description |
|---|---|
| `gc.collect()` | Manually trigger garbage collection, clean up all unreachable objects |
| `gc.mem_free()` | Returns the current free heap memory in bytes |
| `gc.mem_alloc()` | Returns the current allocated heap memory in bytes |
| `gc.enable()` | Enable automatic garbage collection |
| `gc.disable()` | Disable automatic garbage collection (for manual control of GC timing) |
| `gc.isenabled()` | Check if automatic GC is enabled, returns True/False |
| `gc.threshold(n)` | Set the GC trigger threshold (bytes). When cumulative allocation reaches n bytes, automatic GC is triggered. Calling without arguments returns the current threshold. Passing -1 disables the threshold. |

Below is our test code:

```python
# Import the gc module for garbage collection operations (mpy version)
import gc

# 1. Check if automatic GC is enabled (supported by some mpy versions)
print(gc.isenabled())

# 2. Disable automatic GC for manual control of the test process
gc.disable()
print(gc.isenabled())

# 3. View initial free heap memory and allocated heap memory (mpy-specific methods)
print(f"Free heap: {gc.mem_free()} bytes")
print(f"Allocated heap: {gc.mem_alloc()} bytes")

# 4. Demonstrate gc.threshold() method: set and query the GC allocation threshold (mpy-specific method)
# Set threshold to 1024 bytes, trigger GC when cumulative allocation reaches 1024 bytes (early collection reduces fragmentation)
gc.threshold(1024)
# Query current threshold when called without arguments
current_threshold = gc.threshold()  
print(f"Current GC threshold: {current_threshold} bytes")

# 5. Create normal objects, occupying heap memory (variables are references on the stack, pointing to heap objects)
# Create a larger list for more noticeable memory changes
obj1 = [1, 2, 3, 4, 5]  
obj2 = {"name": "test", "age": 18}
print("Objects created")

# 6. View memory changes after creating objects
print(f"Free heap after create: {gc.mem_free()} bytes")
print(f"Allocated heap after create: {gc.mem_alloc()} bytes")

# 7. Cut references, making objects garbage (objects in heap have no references pointing to them)
obj1 = None
obj2 = None
print("References cut, objects become garbage")

# 8. View memory after cutting references (GC not run, memory not freed)
print(f"Free heap before collect: {gc.mem_free()} bytes")
print(f"Allocated heap before collect: {gc.mem_alloc()} bytes")

# 9. Manually trigger GC to clean up garbage objects
# Some mpy versions have no return value, so remove the receiving variable
gc.collect()  
print("Garbage collection executed")

# 10. View memory changes after GC (garbage cleaned up, free memory increases)
print(f"Free heap after collect: {gc.mem_free()} bytes")
print(f"Allocated heap after collect: {gc.mem_alloc()} bytes")

# 11. Demonstrate garbage collection of circular references (reference counting cannot handle this, requires gc.collect())
a = []
b = []
# a references b, b references a, forming a circular reference
a.append(b)  
b.append(a) 
print("Circular reference objects created")

# 12. View memory after creating circular reference objects
print(f"Free heap after circular ref: {gc.mem_free()} bytes")
print(f"Allocated heap after circular ref: {gc.mem_alloc()} bytes")

# 13. Cut external references. Now a and b reference each other, reference count is not zero
a = None
b = None
print("External references cut (circular ref remains)")

# 14. Manually trigger GC to clean up circular reference garbage
gc.collect()
print("Garbage collection for circular ref executed")

# 15. View memory after cleaning up circular references
print(f"Free heap after circular collect: {gc.mem_free()} bytes")
print(f"Allocated heap after circular collect: {gc.mem_alloc()} bytes")

# 16. Restore automatic GC and reset threshold to default (-1 means disable threshold)
gc.threshold(-1)
print(f"Reset GC threshold: {gc.threshold()}")
gc.enable()
print(gc.isenabled())
```

Running the terminal, the output is as follows:

**Figure 12: GC test output (Part 1)**

```
Try the new cross-platform PowerShell https://aka.ms/pscore6

Loading personal and system profiles took 810 milliseconds.
(base) PS G:\test_microMLP> mpremote
Connected to MicroPython at COM65
Use Ctrl-] or Ctrl-x to exit this shell

>>> import gc
>>> print(gc.isenabled())
True
>>> gc.disable()
>>> print(gc.isenabled())
False
>>> print(f"Free heap: {gc.mem_free()} bytes")
Free heap: 227040 bytes
>>> print(f"Allocated heap: {gc.mem_alloc()} bytes")
Allocated heap: 6368 bytes
>>> gc.threshold(1024)
>>> current_threshold = gc.threshold()
>>> print(f"Current GC threshold: {current_threshold} bytes")
Current GC threshold: 1024 bytes
>>> obj1 = [1, 2, 3, 4, 5]
>>> obj2 = {"name": "test", "age": 18}
>>> print("Objects created")
Objects created
>>> print(f"Free heap after create: {gc.mem_free()} bytes")
Free heap after create: 224832 bytes
>>> print(f"Allocated heap after create: {gc.mem_alloc()} bytes")
Allocated heap after create: 8608 bytes
>>> obj1 = None
>>> obj2 = None
>>> print("References cut, objects become garbage")
References cut, objects become garbage
>>> print(f"Free heap before collect: {gc.mem_free()} bytes")
Free heap before collect: 223472 bytes
>>> print(f"Allocated heap before collect: {gc.mem_alloc()} bytes")
Allocated heap before collect: 9968 bytes
>>> gc.collect()
>>> print("Garbage collection executed")
Garbage collection executed
```

**Figure 13: GC test output (Part 2)**

```
Allocated heap before collect: 9968 bytes
>>> gc.collect()
>>> print("Garbage collection executed")
Garbage collection executed
>>> print(f"Free heap after collect: {gc.mem_free()} bytes")
Free heap after collect: 227696 bytes
>>> print(f"Allocated heap after collect: {gc.mem_alloc()} bytes")
Allocated heap after collect: 5744 bytes
>>> a = []
>>> b = []
>>> a.append(b)
>>> b.append(a)
>>> print("Circular reference objects created")
Circular reference objects created
>>> print(f"Free heap after circular ref: {gc.mem_free()} bytes")
Free heap after circular ref: 225904 bytes
>>> print(f"Allocated heap after circular ref: {gc.mem_alloc()} bytes")
Allocated heap after circular ref: 7552 bytes
>>> a = None
>>> b = None
>>> print("External references cut (circular ref remains)")
External references cut (circular ref remains)
>>> gc.collect()
>>> print("Garbage collection for circular ref executed")
Garbage collection for circular ref executed
>>> print(f"Free heap after circular collect: {gc.mem_free()} bytes")
Free heap after circular collect: 227664 bytes
>>> print(f"Allocated heap after circular collect: {gc.mem_alloc()} bytes")
Allocated heap after circular collect: 5792 bytes
>>> gc.threshold(-1)
>>> print(f"Reset GC threshold: {gc.threshold()}")
Reset GC threshold: -1
>>> gc.enable()
>>> print(gc.isenabled())
True
>>>
```

The overall flow is shown in the following diagram:

**Figure 14: GC overall flow diagram**

```
[Initialize GC State]
  GC enabled
  Free memory: 227040B
  Allocated memory: 6368B
  Total memory: ~233KB
       ↓
[Disable automatic GC]
       ↓
[Set GC threshold=1024B]
       ↓
[Create objects obj1, obj2]
  obj1 = [1,2,3,4,5]
  obj2 = {"name":"test","age":18}
  Free ↓2208B → 224832B
  Allocated ↑2240B → 8608B
       ↓
[Cut object references]
  obj1 = None
  obj2 = None
  Objects become garbage
  Free ↓1360B → 223472B
  Allocated ↑1360B → 9968B
       ↓
[Execute manual GC]
  Garbage collection executed
  Free ↑4224B → 227696B
  Allocated ↓4224B → 5744B
  Memory freed: 4224B
       ↓
[Create circular reference]
  a = [], b = []
  a.append(b), b.append(a)
  Circular reference created
  Free ↓1792B → 225904B
  Allocated ↑1808B → 7552B
       ↓
[Cut external references]
  a = None, b = None
  Circular reference island formed
       ↓
[Execute GC to reclaim circular reference]
  Mark-sweep algorithm works
  Free ↑1760B → 227664B
  Allocated ↓1760B → 5792B
  Memory freed: 1760B
       ↓
[Reset GC configuration]
  GC threshold = -1 (automatic management)
  Re-enable GC
```

The memory changes are shown in the following table:

| Operation Phase | Free Heap Memory (B) | Allocated Heap Memory (B) | Description |
|---|---|---|---|
| Initialization | 227040 | 6368 | Total ~233KB |
| Create obj1, obj2 | 224832 | 8608 | Free decreased by 2208B |
| Cut references (GC not run) | 223472 | 9968 | Garbage not reclaimed, memory continues to be consumed |
| After manual GC | 227696 | 5744 | Freed 4224B, memory restored |
| Create circular reference | 225904 | 7552 | Free decreased by 1792B |
| Cut external references (GC not run) | 225904 | 7552 | Island not reclaimed, memory unchanged |
| GC reclaims circular reference | 227664 | 5792 | Mark-sweep algorithm freed 1760B |

We can also use the `micropython.mem_info(1)` method to view the heap utilization table:

**Figure 15: `micropython.mem_info(1)` output**

```
>>> import micropython
>>> micropython.mem_info(1)
stack: 516 out of 7936
GC: total: 233024, used: 7168, free: 225856
 No. of 1-blocks: 96, 2-blocks: 26, max blk sz: 64, max free sz: 14075
GC memory layout; from 200071c0:
00000000: h=MLhhhhDhhh===DhhhDBDBh===BSB=hh====B=BBBBBBBB=BhB=BBBSB=BBhB=Bh
00000400: ===DB=h============h=========================BBBBh==h=hh=========h=======
00000800: ==========h======================================================================
00000c00: ==========h======================================================================
00001000: ==========h====ShShSBhh=h==hhh=Sh=Sh=hh===h====h===hhhh=Bhh=Sh==
00001400: =h=h==h==Bhhh=======h========hSh====h==h==hh=Shhhhhhh=hh==Bhh=h=h
00001800: hh=h===hh===Bh=hhh=hhh==Bh=hhhhhh==B....h=h..h====h====.h....h=h
00001c00: ==.h.................h===..h..hh.B...h.......................
        (219 lines all free)
00038c00: ..............................
```

This is the detailed memory information output of MicroPython, used to view the overall state of the stack and heap. It is the foundation for memory analysis:

```yaml
stack: 516 out of 7936
GC: total: 233024, used: 7168, free: 225856
 No. of 1-blocks: 96, 2-blocks: 26, max blk sz: 64, max free sz: 14075
... (memory layout omitted)
```

- **stack: 516 out of 7936**: Stack memory has used 516 bytes, total size 7936 bytes. Stack memory is used to store local variables and function call contexts. Here, the stack usage is very small and in a safe state.
- **GC: total: 233024, used: 7168, free: 225856**: Core statistics of heap memory (unit: bytes):
  - `total`: Total heap memory size (233024 bytes).
  - `used`: Currently allocated heap memory (7168 bytes), i.e., the heap space occupied by objects in the program.
  - `free`: Currently free heap memory (225856 bytes), available for allocating new objects.
- **Block Information (1-blocks/2-blocks, etc.)**: MicroPython's heap memory is allocated in units of "blocks". This shows the number of blocks of different sizes, the maximum block size, and the maximum free block size, reflecting the degree of heap fragmentation.
  - `No. of 1-blocks: 96`: There are 96 allocated blocks of size 1.
  - `2-blocks: 26`: There are 26 allocated blocks of size 2.
  - `max blk sz: 64`: The largest allocated block size is 64 blocks.
  - `max free sz: 14075`: The largest contiguous free block size is 14075 blocks.
- **Memory Layout**: Displays the usage status of heap memory in character form:
  - `h` indicates a used block (head of an allocation)
  - `=` indicates a free block
  - `D`/`B`/`S`/`M`/`L` are special markers (representing object types like dictionary, bytearray, string, module, etc.)
  - `219 lines all free` indicates that most of the heap memory is free, and the overall memory is healthy.

---

# References

- [MicroPython Official Documentation - Reference Manual](https://docs.micropython.org/en/latest/reference/index.html)
- [08 SPI Serial Peripheral Interface - Document Tutorial Draft](https://f1829ryac0m.feishu.cn/docx/G4P7dNfYso36YLxVlCQc6V4snUc)
- [How to Speed Up MicroPython Execution](https://f1829ryac0m.feishu.cn/docx/BVTZdXCMCobRbFxuM8jcNCEMndc)
