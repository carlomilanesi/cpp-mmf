# Memory-Mapped File C++ Library Tutorial and Reference


## Purpose

This is a library for C++98 language and successive versions, to handle files
as arrays of bytes, exploiting system calls provided by Microsoft Windows
and by POSIX-compliant operating systems.


## Contents

This package is made of one HTML documentation file,
and two C++ files:

* `manual.html`: This document, in HTML format.
* `memory_mapped_file.hpp`: Header file, to be included by every source file
  that needs to read or to write a memory-mapped file.
* `memory_mapped_file.cpp`: Implementation file, to link into the executable.

Only Microsoft Windows and POSIX-compliant operating systems
(like Unix, Linux, and Mac OS X) are supported.
The source files contain several code under conditional compilation.
If the `_WIN32` macro is defined, then the Microsoft Windows API is used;
otherwise the POSIX API is used.

The header file contains only the namespace `memory_mapped_file`,
containing the definition of the following items:

* The `mmf_granularity` function: It allows to get operating system allocation
  granularity (for advanced uses).
* The `base_mmf` abstract class: It contains what is common
  between the other classes. It cannot be instantiated.
* The `read_only_mmf` class: It is used to access an already existing
  file only for reading it.
* The `writable_mmf` class: It is used to access an already existing
  or a not yet existing file for both reading and writing it.
* The `mmf_exists_mode` enumeration: Options for creating a writable
  memory-mapped-file based on an already existing file.
* The `mmf_doesnt_exist_mode` enumeration: Options for creating a writable
  memory-mapped-file based on a not yet existing file.


## Tutorial


### Example

First of all, a complete example of use of the library is presented.
The following program just copies a file.
Put in an empty directory the two library files `memory_mapped_file.hpp` and
`memory_mapped_file.cpp`, and a new file named `example.cpp`,
having the following contents:

    #include "memory_mapped_file.hpp"
    #include <iostream> // for std::cout and std::endl
    #include <algorithm> // for std::copy

    int CopyFile(char const* source, char const* dest, bool overwrite)
    {
        // Create a read-only memory-mapped-file for reading the source file.
        memory_mapped_file::read_only_mmf source_mf(source);

        // Check that the file has been opened.
        if (! source_mf.is_open()) return 1;

        // Check that the contents of the file has been mapped into memory.
        if (! source_mf.data()) return 2;

        // Create a writable memory-mapped-file for writing
        // the destination file, with the option to overwrite it or not,
        // if such file already exists.
        memory_mapped_file::writable_mmf dest_mf(dest,
            overwrite ? memory_mapped_file::if_exists_truncate :
            memory_mapped_file::if_exists_fail,
            memory_mapped_file::if_doesnt_exist_create);

        // Check that the file has been opened.
        if (! dest_mf.is_open()) return 3;

        // Map into memory a (new) portion of the file,
        // as large as the source file.
        dest_mf.map(0, source_mf.file_size());

        // Check that the contents of the file has been mapped into memory.
        if (! dest_mf.data()) return 4;

        // Check that the source buffer has the same size
        // of the destination buffer. It cannot be otherwise.
        if (source_mf.mapped_size() != dest_mf.mapped_size()) return 5;

        // Check that the source file has the same size
        // of the destination file. It cannot be otherwise.
        if (source_mf.file_size() != dest_mf.file_size()) return 6;

        // Copy the source buffer to the destination buffer.
        std::copy(source_mf.data(), source_mf.data() + source_mf.mapped_size(),
            dest_mf.data());
        return 0;
    }

    int main()
    {
        using namespace std;

        // Copy the first file, overwriting the second file,
        // if it already exists.
        // It should print always 0, meaning success.
        cout << CopyFile("memory_mapped_file.hpp", "copy.tmp", true) << endl;

        // Copy the first file to the second file,
        // but only if the second file does not already exist.
        // It should print always 3, meaning failure,
        // as here the second file already exists.
        cout << CopyFile("memory_mapped_file.hpp", "copy.tmp", false) << endl;
    }

To compile and run the example, in a Windows environment
with Visual C++ installed, from a command prompt, type:

    cl /nologo /EHsc example.cpp memory_mapped_file.cpp /Feexample.exe

and then

    example.exe

Instead, in a POSIX environment with GCC installed, from a shell, type:

    g++ example.cpp memory_mapped_file.cpp -o example

and then

    ./example

In both environments the program should print,
even if it is run several times:

    0
    3

and should create a file named `copy.tmp`, identical
to the file `memory_mapped_file.hpp`.

The behavior of this example is explained in the comments embedded
in the file `example.cpp`.

But now we'll do a step-by-step tutorial.


### Set-up

In this tutorial, we'll write several versions of a file named `tutorial.cpp`.
The first version has the following contents:

    #include "memory_mapped_file.hpp"
    using namespace memory_mapped_file;
    #include <fstream>
    #include <iostream>
    #include <string>
    using namespace std;

    char const pathname[] = "a.txt";

    void create_file()
    {
        ofstream f(pathname);
        f << "Hello, world!";
        f.close();
    }

    int main()
    {
    }

To compile it using Visual C++, type the following command line:

    cl /nologo /EHsc memory_mapped_file.cpp tutorial.cpp /Fetutorial.exe

To compile it using using GCC, type the following command line:

    g++ memory_mapped_file.cpp tutorial.cpp -o tutorial


Of course, this program does nothing, as it has an empty `main` function.
The following versions will change only the body of the `main` function.

The `create_file` function creates in the current directory a file
named `a.txt`, containing only the 13 bytes `Hello, world!`.


### Opening and mapping a whole file

Write the following contents for the `main` function
of the `tutorial.cpp` file:

        cout << boolalpha;
        create_file();
        read_only_mmf mmf(pathname);
        cout << mmf.is_open() << endl;
        cout << mmf.file_handle() << endl;
        cout << mmf.file_size() << endl;
        cout << mmf.offset() << endl;
        cout << mmf.mapped_size() << endl;

When it is run, it should print:

    true
    <OS dependent value>
    13
    0
    13

The first statement ensures that `bool` expressions are printed as
`true` or `false`.

The second statement ensures that there is a data file to use.

The third statement defines and initializes an object owning
a memory-mapped-file. The constructor of such object tries to open
the specified file (searching it from the current directory)
and to map its contents in its address space.

Presumably it finds such file and can open it, and therefore
the call to `is_open` should return `true`.

The call to `file_handle` should return an operating-system-dependent value
to handle the underlying file.
For example, it can be used to lock the file for exclusive use,
using operating system calls.

The call to `file_size` should return the length of the opened file, i.e. `13`.

By default, the whole contents of the file is mapped to memory.
Therefore the call to `offset`, that returns the starting point
of the mapping inside the file, should return `0`,
i.e. there is no offset.

As the whole file is mapped, the call to `mapped_size`,
that returns the length of the mapped portion of the file,
should return the length of the file, i.e. `13`.


### Failing to open a file

Now replace the third line of the `main` function with the following one,
where `pathname` is replaced by `"x"`:

        read_only_mmf mmf("x");

When it is run, it should print:

    false
    <OS dependent value>
    0
    0
    0

Now the call to `is_open` returns `false`, as the specified file
cannot be opened.
The call to `file_handle` returns an operating-system-dependent value
representing an invalid file handle.
The last three function calls return `0`,
as the underlying file has not been opened.

The `read_only_mmf` class contains no function to open or close
explicitly a file.
The underlying file can be opened only by the constructor,
using the specified pathname.
If the opening of such file fails, the object becomes useless.
The `is_open` function is useful to check if the object initialization
was successful.
The underlying file is closed only implicitly by the destructor.


### Mapping and un-mapping

Now replace all the contents of the `main` function with the following lines:

        cout << boolalpha;
        create_file();
        read_only_mmf mmf(pathname, false);
        cout << mmf.is_open() << endl;
        cout << mmf.file_size() << endl;
        cout << mmf.offset() << endl;
        cout << mmf.mapped_size() << endl;
        mmf.map(2, 6);
        cout << mmf.is_open() << endl;
        cout << mmf.file_size() << endl;
        cout << mmf.offset() << endl;
        cout << mmf.mapped_size() << endl;
        mmf.unmap();
        cout << mmf.is_open() << endl;
        cout << mmf.file_size() << endl;
        cout << mmf.offset() << endl;
        cout << mmf.mapped_size() << endl;

When it is run, it should print:

    true
    13
    0
    0
    true
    13
    2
    6
    true
    13
    0
    0

By passing to the constructor the optional parameter `false`,
the file is opened but not mapped.
Therefore, in his example, although the `is_open` function always returns
`true`, and the `file_size` function always returns the correct file size,
the first call to `mapped_size` returns `0`, as no mapping is done.

The call to the `map` member function creates a mapping from the offset
specified by the first argument, for the length specified
by the second argument.
This appears also by the ensuing calls to `offset` and `mapped_size`, that
return `2` and `6`, respectively.

Then, by calling the `unmap` member function, the mapping is undone.


### More on mapping

Now replace all the contents of the `main` function with the following lines:

        create_file();
        read_only_mmf mmf(pathname, false);
        mmf.map(2);
        cout << mmf.offset() << endl;
        cout << mmf.mapped_size() << endl;
        mmf.map();
        cout << mmf.offset() << endl;
        cout << mmf.mapped_size() << endl;
        mmf.map(10, 10000);
        cout << mmf.offset() << endl;
        cout << mmf.mapped_size() << endl;

When it is run, it should print:

    2
    11
    0
    13
    10
    3

First, notice that the second call to `map` is performed without
before calling `unmap`. Actually `unmap()` is implicitly called
by the `map` function and by the destructor.

Then, notice that `map` may have only one argument or no arguments,
as they have both the default value `0`.

Then, notice that the `0` value for the second argument of `map` doesn't mean
that the mapping will have zero length (that is impossible),
but that it will extend up to the length of the file, if possible.

Then, notice that even if the specified offset plus the specified length
exceeds the length of the file, the mapping extends anyway up to the length
of the file, as shown by the last printed line.


### Reading the file contents

Now replace all the contents of the `main` function with the following lines:

        create_file();
        read_only_mmf mmf(pathname);
        cout << string(mmf.data(), 8) << endl;
        mmf.map(2);
        cout << string(mmf.data(), 8) << endl;
        mmf.map(3000);
        cout << (size_t)mmf.data() << endl;

When it is run, it should print:

    Hello, w
    llo, wor
    0

If the `map` call succeeds, every ensuing calls to `data` return
a pointer to the mapped buffer starting at the specified offset.

But the `map` function may fail, for several reasons.
For example, no mapping is possible if the file is not opened,
or if the specified offset is equal to or greater than the file size,
or if there is not enough address space to map all the specified range.

If the `map` call fails, every ensuing call to `data` returns a null pointer;
therefore, every time you call `map`, you should check the value returned
by `data` or by `mapped_size`, before dereferencing the pointer
returned by `data`.


### Mapping granularity

Operating systems do not allow to map files to memory starting from every
specified byte. They require that the offset be a multiple of a number,
named _granularity_, that is dependent on the operating system,
and typically may vary from 4 KB to 64 KB.

To avoid bothering users with such technicality, this library takes care
of mapping memory internally from the nearest boundary.
For example, if the granularity is 65536, and for a 10 MB long file a mapping
is requested from 500000 to 800000,
the memory actually mapped by the operating system is from 458752
to a number somewhat greater than 800000,
but the `offset` function will return 500000,
and the `mapped_size` function will return 300000.

To allow the user to take granularity into account,
there is a global function named `mmf_granularity`,
that returns such granularity size.


### Read-only access

Now replace all the contents of the `main` function with the following lines:

        create_file();
        read_only_mmf mmf(pathname, false);
        cout << mmf.data()[0];
        mmf.data()[0] = 'a';

When it is compiled, a syntax error should occurs in the last statement,
as the call to `data` returns a `const` address.
However, if such const-ness is bypassed using a cast,
the program will compile, but that statement will generate a run-time error,
as the operating system has marked such address range as _read-only_.


### Opening for writing

Up to now, only the `read_only_mmf` class has been used.
Such class disallows to change the file.

Memory mapped files may be used also for changing the contents of files,
as the following examples show.

Now replace all the contents of the `main` function with the following lines:

        cout << boolalpha;
        create_file();
        writable_mmf mmf(pathname, if_exists_map_all, if_doesnt_exist_fail);
        cout << mmf.is_open() << endl;
        cout << mmf.file_size() << endl;
        cout << mmf.offset() << endl;
        cout << mmf.mapped_size() << endl;
        cout << string(mmf.data(), 8) << endl;

When it is run, it should print:

    true
    13
    0
    13
    Hello, w

It means that the file has been opened, its size is 13 bytes,
all the file has been mapped, and its first `8` characters are `Hello, w`.

As it appears, this object can be used just like a read-only memory mapped
file, except for the constructor and except for something we'll see later.

Actually, most that can be done using a `read_only_mmf` object
can also be done using a `writable_mmf` object.

Of course, when a file is to be created or changed, it is required to use
the `writable_mmf` class.
Instead, when a file is only to be read, both classes could be used.
Nevertheless, using `read_only_mmf` has the following advantages:

* **Reading read-only files or files shared only for reading**:
  `writable_mmf` objects require to open the specified file
  in read/write mode, and the operating system prevents such operation
  on files marked as *read-only* or shared only for reading.
  The only way to read a read-only file or a file shared only for reading
  is to use a `read_only_mmf` object.
* **Simpler API**: As `read_only_mmf` cannot change to specified file,
  it has fewer features, and therefore it is simpler to learn and use.
* **No risk of accidental change**: As `writable_mmf` objects
  open the specified file in read/write mode, a logically erroneous operation
  or an undefined behavior operation could apply unwanted changes
  to the contents of such file.
  As `read_only_mmf` objects open the specified file in read-only mode,
  the operating system prevents any subsequent attempt to change it.
* **Possibly more efficient**: Operating systems may use more efficient
  buffering algorithms for read-only files, used by `read_only_mmf` objects,
  than for read/write files, used by `writable_mmf` objects.


### Read/write mapping mode

We have already seen one usage instance of the `writable_mmf` class.
There are several other ways to create such object,
described in the reference.
The four most typical ones are used in the program
whose `main` function has the following contents:

        cout << boolalpha;

        // 1. Fail or create
        { // 1.1. File exists
            create_file();
            writable_mmf mmf(pathname,
                if_exists_fail, if_doesnt_exist_create);
            cout << mmf.is_open() << endl;
        }
        { // 1.2. File doesn't exist
            remove(pathname);
            writable_mmf mmf(pathname,
                if_exists_fail, if_doesnt_exist_create);
            cout << mmf.is_open() << " " << mmf.file_size() << endl;
        }

        // 2. Truncate or create
        { // 2.1. File exists
            create_file();
            writable_mmf mmf(pathname,
                if_exists_truncate, if_doesnt_exist_create);
            cout << mmf.is_open() << " " << mmf.file_size() << endl;
        }
        { // 2.2. File doesn't exist
            remove(pathname);
            writable_mmf mmf(pathname,
                if_exists_truncate, if_doesnt_exist_create);
            cout << mmf.is_open() << " " << mmf.file_size() << endl;
        }

        // 3. Just open or fail
        { // 3.1. File exists
            create_file();
            writable_mmf mmf(pathname,
                if_exists_just_open, if_doesnt_exist_fail);
            cout << mmf.is_open() << " " << mmf.file_size() << " "
                << mmf.mapped_size() << endl;
        }
        { // 3.2. File doesn't exist
            remove(pathname);
            writable_mmf mmf(pathname,
                if_exists_just_open, if_doesnt_exist_fail);
            cout << mmf.is_open() << endl;
        }

        // 4. Map all or fail
        { // 4.1. File exists
            create_file();
            writable_mmf mmf(pathname,
                if_exists_map_all, if_doesnt_exist_fail);
            cout << mmf.is_open() << " " << mmf.file_size() << " "
                << mmf.mapped_size() << endl;
        }
        { // 4.2. File doesn't exist
            remove(pathname);
            writable_mmf mmf(pathname,
                if_exists_map_all, if_doesnt_exist_fail);
            cout << mmf.is_open() << endl;
        }

When it is run, it should print:

    false
    true 0
    true 0
    true 0
    true 13 0
    false
    true 13 13
    false

There are 4 cases, each one with two sub-cases:
in the first one the file already exists,
and in the second one the file doesn't exist yet.

In case 1.1., the file exists, and there is the option `if_exists_fail`;
therefore such file is not opened, and `is_open` returns `false`.

In case 1.2., the file does not exist, and there is the option
`if_doesnt_exist_create`;
therefore such file is created empty, and so `is_open` returns `true`,
and `file_size` returns `0`.

Case 1 ("Fail or create") is useful when it is needed to create a file,
without overwriting an existing file.

In case 2.1., the file exists, and there is the option `if_exists_truncate`;
therefore such file is opened and truncated,
and so `is_open` returns `true`, and `file_size` returns `0`.

In case 2.2., the file does not exist, and there is the option
`if_doesnt_exist_create`;
therefore such file is created empty, and so `is_open` returns `true`,
and `file_size` returns `0`.

Case 2 ("Truncate or create") is useful when it is needed to create a file,
even overwriting an existing file.

In case 3.1., the file exists, and there is the option `if_exists_just_open`;
therefore such file is opened and not truncated nor mapped,
and so `is_open` returns `true`, `file_size` returns `13`,
and `mapped_size` returns `0`.

In case 3.2., the file does not exist, and there is the option
`if_doesnt_exist_fail`;
therefore such file is not opened, and so `is_open` returns `false`.

Case 3 ("Just open or fail") is useful when it is needed
to change a very large existing file, to be mapped piece-wise.

In case 4.1., the file exists, and there is the option `if_exists_map_all`;
therefore such file is opened and not truncated, and it is mapped all,
and so `is_open` returns `true`, `file_size` returns `13`,
and `mapped_size` returns `13`.

In case 4.2., the file does not exist, and there is the option
`if_doesnt_exist_fail`;
therefore such file is not opened, and so `is_open` returns `false`.

Case 4 ("Map all or fail") is useful when it is needed
to change a not-so-large existing file, to be handled as a single string.


### Read/write access

Up to now, in our examples, no file has been changed by a memory-mapped-file.

Now replace all the contents of the `main` function with the following lines:

        create_file();
        {
            writable_mmf mmf(pathname,
                if_exists_map_all, if_doesnt_exist_fail);

            cout << string(mmf.data(), 6) << endl;
            mmf.data()[1] = 'X';
            cout << string(mmf.data(), 6) << endl;
        }
        {
            read_only_mmf mmf(pathname);
            cout << string(mmf.data(), 6) << endl;
        }

It should print:

    Hello,
    HXllo,
    HXllo,

This means that the `data` function of the `writable_mmf` class,
returns a non-`const` address of a buffer,
and when some bytes of such buffer are changed,
such changes are immediately visible to the process, and are also
saved to the file sometime no later than when the current scope is closed.


### Flushing writes

Such effective write to the file is handled by the operating system,
and, for efficiency reasons, usually it does not happen immediately,
as shown by replacing all the contents of the `main` function
with the following lines:

        create_file();
        writable_mmf mmf(pathname,
            if_exists_map_all, if_doesnt_exist_fail);
        mmf.data()[0] = 'X';
        mmf.flush();
        mmf.data()[1] = 'Y';
        cin.get();

This program writes a letter "X" as the first byte of the file,
and calls the `flush` function to ensure it is written to the file.
Then it writes a letter "Y" as the second byte of the file,
and waits for user input.
If you now press the reset button of your computer,
or remove any electric power supply, preventing the operating system to save
this second byte to safe storage, and then restart your computer,
and look into file "a.txt", you should find it has the following contents:

    Xello, world!"

As you can see, the `Y` character has not been written to the file.

This brutal procedure is necessary to show the effect,
because if you terminate the process in any other way,
the operating system is still able to save the buffer to persistent storage.

The `flush` call, albeit more efficient that closing and reopening the file,
is rather inefficient, though, because it writes data to physical storage,
and so it should be used only when data consistency is required
even in case of a power failure or an operating system crash.


# Reference

## The `mmf_granularity` function

Scope: namespace `memory_mapped_file`.


## The `base_mmf` class

Scope: namespace `memory_mapped_file`.

Abstract class representing a memory-mapped-file.
It is the base class of `read_only_mmf` and `writable_mmf`.

Its possible states are:

1. File not opened.
1. File opened but not mapped.
1. File opened and mapped.

The constructor sets the object in one of the three possible states.

The file can be opened only by the constructor.
If the constructor fails to open the file, the object is useless,
as it cannot open the file any more.

The constructor can open without mapping the file,
or open and map the file.

Then the file can be mapped using the function `map`,
or 

### The function `base_mmf()`

The only constructor.

### The function `~base_mmf()`

The destructor.

### The function `size_t offset() const`

Scope: class `base_mmf`.

It returns the distance in bytes from the beginning of the file
to the beginning of the portion of the file currently
mapped to memory.
Therefore, it returns `0` (zero) when the mapping starts at the beginning
of the file.
It returns `0` also when the file hasn't been opened, or when it is open
but there is no mapping.

### The function `size_t mapped_size() const`

It returns the size in bytes of the portion of the file currently
mapped to memory.
It returns `0` when and only when the file hasn't been opened,
or when it is open but there is no mapping.

### The function `size_t file_size() const`

It returns the whole size of the underlying opened file.
Of course, it returns `0` when the opened file has zero-length,
but also when the file hasn't been opened.

### The function `void unmap()`

It cancels the current mapping.

### The function `bool is_open() const`

It returns `true` if and only if the underlying file is mapped to memory.

### The type name `HANDLE`

Such name represents the operating-system-dependent type
of the handle of a file.
For Microsoft Windows, it is a `void*`; for POSIX systems, it is `int`.

### The function `HANDLE file_handle() const`

It returns the operating-system-dependent handle used internally to access
the file.
It may be used to access operating-system-dependent operations,
not defined by this library, like file locking.

## The `read_only_memory_mapped_file` class

### `explicit read_only_memory_mapped_file(char const* pathname)`

### `char const* data() const`

It returns the address of the beginning of the memory buffer mapped
to a portion of the file.

### `char const* map(size_t offset = 0, size_t size = 0)`

It creates a mapping between a portion of the file and a memory buffer.


## The `read_write_memory_mapped_file` class

### `enum e_open_mode`

A `read_write_memory_mapped_file` must be created in one of the following
modes:

* `if_exists_fail_else_create`: If the specified file was already existing,
  nothing is done; otherwise it is created empty.
* `if_exists_keep_else_fail`: If the specified file was already existing,
  it is opened as such; otherwise nothing is done.
* `if_exists_keep_else_create`: If the specified file was already existing,
  it is opened as such; otherwise it is created empty.
* `if_exists_truncate_else_fail`: If the specified file was already existing,
  it is opened and truncated to zero size; otherwise nothing is done.
* `if_exists_truncate_else_create`: If the specified file was already existing,
  it is opened and truncated to zero size; otherwise it is created empty.

### `explicit read_write_memory_mapped_file(char const* pathname, e_open_mode open_mode)`

    #include "memory_mapped_file.hpp"
    #include <iostream>
    using namespace std;

    int main()
    {
        {
            // If the file already exists, it is not opened;
            // otherwise, it is created empty.
            read_write_memory_mapped_file mmf("a.txt",
                read_write_memory_mapped_file::
                    if_exists_fail_else_create);
        }
        {
            // If the file already exists, it is not opened;
            // otherwise, it is created with a size of 60 bytes,
            // and it is mapped all.
            read_write_memory_mapped_file mmf("a.txt",
                read_write_memory_mapped_file::
                    if_exists_fail_else_create, 60);
        }
        {
            // If the file already exists, it is opened, but not mapped;
            // otherwise, it is not created.
            read_write_memory_mapped_file mmf("a.txt",
                read_write_memory_mapped_file::
                    if_exists_keep_else_fail);
        }
        {
            // If the file already exists, it is opened, and mapped all;
            // otherwise, it is not created.
            // For the last argument it is relevant only
            // to be zero or non-zero.
            read_write_memory_mapped_file mmf("a.txt",
                read_write_memory_mapped_file::
                    if_exists_keep_else_fail, 60);
        }
        {
            // If the file already exists, it is opened, but not mapped;
            // otherwise, it is created empty.
            read_write_memory_mapped_file mmf("a.txt",
                read_write_memory_mapped_file::
                    if_exists_keep_else_create);
        }
        {
            // If the file already exists, it is opened, and mapped all;
            // otherwise, it is created with a size of 60 bytes,
            // and it is mapped all.
            read_write_memory_mapped_file mmf("a.txt",
                read_write_memory_mapped_file::
                    if_exists_keep_else_create, 60);
        }
        {
            // If the file already exists, it is opened
            // and truncated to zero length;
            // otherwise, it is not created.
            read_write_memory_mapped_file mmf("a.txt",
                read_write_memory_mapped_file::
                    if_exists_truncate_else_fail);
        }
        {
            // If the file already exists, it is opened
            // and truncated to zero length;
            // otherwise, it is not created.
            // This behavior is the same as the previous case,
            // as the last argument is ignored.
            read_write_memory_mapped_file mmf("a.txt",
                read_write_memory_mapped_file::
                    if_exists_truncate_else_fail, 60);
        }
        {
            // If the file already exists, it is opened
            // and truncated to zero length;
            // otherwise, it is created empty.
            read_write_memory_mapped_file mmf("a.txt",
                read_write_memory_mapped_file::
                    if_exists_truncate_else_create);
        }
        {
            // If the file already exists, it is opened
            // and truncated to zero length;
            // otherwise, it is created with a size of 60 bytes,
            // and it is mapped all.
            read_write_memory_mapped_file mmf("a.txt",
                read_write_memory_mapped_file::
                    if_exists_truncate_else_create, 60);
        }
    }

### `char* data()`
### `char* map(size_t offset = 0, size_t size = 0)`


### `bool flush()`

It copies all the changes to the file system, ensuring that
they are persistent.

Actually, when a byte is modified in a memory-mapped file, that change
may be applied much later to the underlying file, possibly only when
the memory-mapped file is closed.

To ensure that every change is actually applied to the storage device,
it is possible to close and reopen the memory-mapped file.
The `flush` operation achieves the same effect much more efficiently.

If returns `true` if the operation succeeds, otherwise `false`.
