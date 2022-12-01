---
layout: post
title:  "A brief overview of Pythons import system"
date:   2022-11-29 22:15:28 +0100
categories: python
---

One of the most poorly understood parts of Python (if we take the average developer) has got to be its import system.
This is partly because `import thingy` *usually* just works™ and partly because a lot of programming literature tends to focus more on fancy syntax features and less on grueling details that make large software projects with many dependencies *actually work*.
So let's take a (very shallow) dive into these grueling details today.

Note that I use Python 3.10.6 and pyenv so your output paths might be different from mine.

## The basics

In case you have been asleep for the past few years of your Python career (in which case I sympathize), here is a recap of the basics.

**Importing** is formally defined as *the process by which code in one module is made available to another module*.
We can import modules using the `import` statement which does two things.
First it searches for the respective module by invoking `__import__`.
Second it assigns the result of `__import__` to a name in the local scope.

Let's say we have a file `flunkifier.py` in a directory `somedir` with the following content:

```python
def flunkify():
    print("all the flunkies have been flunkified")
```

If we start a REPL in the directory `somedir` or create a Python file in `somedir`, we can import the `flunkifier` module from the REPL or the file by simply executing the statement `import flunkifier`.
This will find the `flunkifier` module and assign the result to the name `flunkifier`.
Afterwards we can call `flunkifier.flunkify()` just like you would expect:

```
>>> import flunkifier
>>> flunkifier.flunkify()
all the flunkies have been flunkified
```

Alternatively we could call `__import__` manually.
This is rarely useful, but it highlights that the name of the module does not need to be equal to the name it is assigned to.
For example we could search for the module `flunkifier` and assign it the name `flunkifier2`:

```
>>> flunkifier2 = __import__("flunkifier", globals(), locals(), [], 0)
>>> flunkifier2.flunkify()
all the flunkies have been flunkified
```

Additionally, modules can be grouped in packages.
For example we could create a directory `flunky` which contains the module `flunkifier`.
In this case we can import flunkifier by doing `import flunky.flunkifier`:

```
>>> import flunky.flunkifier
>>> flunky.flunkifier.flunkify()
all the flunkies have been flunkified
```

Alternatively we can use the `from ... import` statement:

```
>>> from flunky import flunkifier
>>> flunkifier.flunkify()
all the flunkies have been flunkified
```

We can also use aliases:

```
>>> import flunky.flunkifier as ff
>>> ff.flunkify()
all the flunkies have been flunkified
>>> from flunky import flunkifier as f
>>> f.flunkify()
all the flunkies have been flunkified
```

This is easy enough, but things rapidly become more complicated as we realize that in a real software project we are rarely in the same directory as the module we wish to import.
Therefore the first big question we need to answer is: *How does Python find the location of a module given its name?*

## Setting the stage

To better visualize what's going on, we will create an example package `mypackage` inside our directory `somedir`.
This package doesn't do anything useful and its sole purpose is provide us with examples for this writeup.

And here is the structure of the `somedir` directory with the `mypackage` package:

```
somedir/
└── mypackage
    ├── flunky
    │   ├── __init__.py
    │   ├── subflunky1
    │   │   ├── flunkifier.py
    │   │   └── __init__.py
    │   └── subflunky2
    │       ├── flunkifier.py
    │       └── __init__.py
    ├── __init__.py
    └── thingy
        ├── __init__.py
        └── thingier.py
```

Here are the contents of the respective Python files:

```
> somedir/mypackage/__init__.py
print("mypackage.__init__ executed")

> somedir/mypackage/flunky/__init__.py
print("flunky.__init__ executed")

> somedir/mypackage/flunky/subflunky1/__init__.py
print("flunky.subflunky1.__init__ executed")

> somedir/mypackage/flunky/subflunky1/flunkifier.py
def flunkify1():
    print("flunkify using algorithm 1")

print("flunky.subflunky1.flunkifier executed")

> somedir/mypackage/flunky/subflunky2/__init__.py
print("flunky.subflunky2.__init__ executed")

> somedir/mypackage/flunky/subflunky2/flunkifier.py
def flunkify2():
    print("flunkify using algorithm 2")

print("flunky.subflunky2.flunkifier executed")

> somedir/mypackage/thingy/__init__.py
print("thingy.__init__ executed")

> somedir/mypackage/thingy/thingier.py
def thingify():
    print("thingify")

print("thingy.thingier executed")
```

I suggest you recreate this structure somewhere on your machine and follow along.

## Searching for a module

The only thing we give the `import` statement is the fully qualified name of the module (like `math` or `flunky.flunkify`).
From here the import system miraculously (usually) finds the correct module regardless of whether its a built-in C module, a built-in Python module, a module in some deeply nested directory or a module in a third-party package.
Let's unravel the magic behind that.

Common wisdom (even among experienced Python programmers) is that Python searches for modules in `sys.path`.
This is *wrong* (or at least way too oversimplified to be a useful mental model).

The reality looks like this:
When a module is imported, the import system has a look at `sys.modules`.
This is just a dictionary that contains the modules that have been already loaded.
If a module is present there, Python simply hands us that module.
This has one relatively important practical consequence.

Consider the `mypackage` package.
This contains some setup code in the `__init__.py` file.
When we import `mypackage` for the very first time, that code is run resulting in the line `mypackage.__init__ executed` printed to the output.
However if we import `mypackage` a second time, it will already be in `sys.modules` and the `__init__.py` will *not* be executed again:

```
>>> import sys
>>> "mypackage" in sys.modules
False
>>> import mypackage
mypackage.__init__ executed
>>> "mypackage" in sys.modules
True
>>> import mypackage
```

If the module is not in `sys.modules` - as is usually the case - Python will tell the (quite appropriately named) **finders** to find that module.
Note that a finder does not give us the module itself.
Instead it hands us a loader for the module, which is then responsible for *actually loading* the module.
This indirection is one of the (many) things that can become confusing when trying to debug problems related to the import system.

The most important finders are the **meta path finders**.
These are the finders contained in `sys.meta_path` list.
Here is how that list looks on my machine:

```
>>> import sys
>>> for meta_finder in sys.meta_path:
...     print(meta_finder)
... 
<_distutils_hack.DistutilsMetaFinder object at 0x7f1ac4b1b910>
<class '_frozen_importlib.BuiltinImporter'>
<class '_frozen_importlib.FrozenImporter'>
<class '_frozen_importlib_external.PathFinder'>
```

Each of these meta path finders has a `find_spec` function which takes the module name and a bunch of other arguments (that we ignore for now).
If the finder knows how to find the module, `find_spec` returns a `ModuleSpec` object describing how to load that module.
Otherwise `find_spec` returns `None`.

Here is a function that returns the finder object that would find the given module: 

```python
def get_finder(module_name):
	for finder in sys.meta_path:
		if finder.find_spec(module_name, None):
			return finder
	return None
```

Let's ignore the `DistutilsMetaFinder` and the `FrozenImporter` for now and focus on the `BuiltinImporter` and the `PathFinder`.
The `BuiltinImporter` knows how to find modules that are directly compiled into the interpreter.
Note that this is the minority of the modules in the standard library.
You can get all such modules via `sys.builtin_module_names`:

```
>>> sys.builtin_module_names
('_abc', '_ast', '_codecs', '_collections', '_functools', '_imp', '_io', '_locale', '_operator', '_signal', '_sre', '_stat', '_string', '_symtable', '_thread', '_tracemalloc', '_warnings', '_weakref', 'atexit', 'builtins', 'errno', 'faulthandler', 'gc', 'itertools', 'marshal', 'posix', 'pwd', 'sys', 'time', 'xxsubtype')
```

For example the `builtins` module is found by the `BuiltinImporter` since `builtins` is directly compiled into the interpreter.
On the other hand, the `math` module is part of the standard library, but is not directly compiled into the interpreter and will therefore be found by the `PathFinder` and *not* the `BuiltinImporter`.
Our custom `mypackage` module will also be found by the `PathFinder`.

Here are the outputs of `get_finder` for the respective modules:

```
>>> get_finder("builtins")
<class '_frozen_importlib.BuiltinImporter'>
>>> get_finder("math")
<class '_frozen_importlib_external.PathFinder'>
>>> get_finder("mypackage")
<class '_frozen_importlib_external.PathFinder'>
```

In summary, the sequence of steps Python takes to a find a module (or more precisely to find the loader for a module) is as follows:

1. Check if the module is already present in `sys.modules`.
2. Iterate over all meta path finders and check if one of them knows how to import the module.

The most important part of the second step is the **path based finder** (which is the `PathFinder` object we saw earlier).
This is also the reason why most people assume that importing works by walking over `sys.path`, because this one of the things the path based finder does.
However, as well will see shortly, there is more to the path based finder than just iterating over `sys.path`.

## The path based finder

The path based finder searches an **import path** which contains a list of **path entries**.
Each path entry names a location to search for modules.
Typically the import path is indeed `sys.path` (but see below for an exception).

Here is how `sys.path` looks on my machine when I start a REPL:

```
>>> for p in sys.path:
...     print(p)
... 

/home/uhasker/.pyenv/versions/3.10.6/lib/python310.zip
/home/uhasker/.pyenv/versions/3.10.6/lib/python3.10
/home/uhasker/.pyenv/versions/3.10.6/lib/python3.10/lib-dynload
/home/uhasker/.local/lib/python3.10/site-packages
/home/uhasker/.pyenv/versions/3.10.6/lib/python3.10/site-packages
```

Yes, the first item is a `zip` file.
This is because the `zipimport` module enables us to import Python modules from ZIP archives which can come in handy in certain situations.
However, in this writeup we will ignore that capability of the import system.
It's just a brief guide after all.

The path based finder moves through every path entry in the import path and associated each path entry with a **path entry finder**.

> Don't be confused.
The path based finder is one of the meta path finders.
A path entry finder is a finder which knows how to locate modules given a specific path entry.
These two things are completely different (even though they have similar names).
Think of path entry finders as an implementation detail of the path based finder.

The path based finder maintains a cache in the `sys.path_importer_cache` dictionary which maps path entries to path entry finders:

```
>>> for k, v in sys.path_importer_cache.items():
...     print(f"{k} : {v}")
... 
/home/uhasker/.pyenv/versions/3.10.6/lib/python310.zip : None
/home/uhasker/.pyenv/versions/3.10.6/lib/python3.10 : FileFinder('/home/uhasker/.pyenv/versions/3.10.6/lib/python3.10')
# this goes on
```

If we find the path entry we are currently looking at in the cache, we are home free.
If we don't, we need to look at `sys.path_hooks` which contaings the so called **path entry hooks**.
Each of these hooks is a callable that can be called with the path entry as the single argument.
This callable returns either the path entry finder (if it knows how to handle the path entry) or raises an `ImportError`.

```
>>> sys.path_hooks
[<class 'zipimport.zipimporter'>, <function FileFinder.path_hook.<locals>.path_hook_for_FileFinder at 0x7fb54608caf0>]
>>> sys.path_hooks[0]("mypackage")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<frozen zipimport>", line 89, in __init__
zipimport.ZipImportError: not a Zip file
>>> sys.path_hooks[1]("mypackage")
FileFinder('/home/uhasker/somedir/mypackage')
```

If no path entry hook hands us a path entry finder, then the path based finder tells the import system that it can't find the module.

Once the path entry finder is found, the `find_spec` method is called with the name of the module being imported:

```
>>> finder = sys.path_hooks[1]("mypackage")
>>> finder.find_spec("flunky")
ModuleSpec(name='flunky', loader=<_frozen_importlib_external.SourceFileLoader object at 0x7f1ac47be8f0>, origin='/home/uhasker/somedir/mypackage/flunky/__init__.py', submodule_search_locations=['/home/uhasker/somedir/mypackage/flunky'])
```

To summarize, here is how the path based finder works:

1. Iterate over the import path (which will *usually* be `sys.path`) and for each path entry:
	* check if the path entry is present in `sys.path_importer_cache`
	* check if a hook in `sys.path_hooks` can give us a path entry finder 
2. Once we have a path entry finder call its `find_spec` method to obtain a module spec.

## Loading and executing

If the finders were able to finde the module, we know have a `ModuleSpec` object returned by the respective finders `find_spec` method.
This object has a `loader` attribute which contains the **loader** that can be used to load the module.

```
>>> spec = sys.meta_path[-1].find_spec("math", None)
>>> module = spec.loader.create_module(spec)
>>> module
<module 'math'>
```

After the module is loaded we add it to `sys.modules`:

```
>>> sys.modules[spec.name] = module
>>> sys.modules["math"]
<module 'math'>
```

Finally we execute the module:

```
>>> spec.loader.exec_module(module)
```

There are many edge cases and legacy things we glossed over here, but you usually don't need to know them.

## Some notes on packages

As we have already discussed, to better organize modules, Python has the concept of packages.
Usually packages are directories and modules files within these directories, but this doesn't need to always be the case.
Technically, **packages** are just a special kind of module (namely modules which have a  `__path__` attribute).

```
>>> import mypackage.thingy
>>> import mypackage.thingy.thingier
>>> type(mypackage.thingy)
<class 'module'>
>>> type(mypackage.thingy.thingier)
<class 'module'>
>>> mypackage.thingy.__path__
['/home/uhasker/somedir/mypackage/thingy']
>>> mypackage.thingy.thingier.__path__
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: module 'mypackage.thingy.thingier' has no attribute '__path__'. Did you mean: '__name__'?
```

Generally speaking, there is only one type of module object (it doesn't matter if the module is a package or something written in C).

There are **regular packages** and **namespace packages**.
A regular package is essentially a directory with an `__init__.py` file.
When a regular package is imported, this file is executed.
Namespace packages are essentially everything else and we will not discuss them here.

If we import a subpackage, *all* parent packages are imported as well:

```
>>> sys.modules.get("mypackage", None) 
>>> sys.modules.get("mypackage.flunky", None) 
>>> sys.modules.get("mypackage.flunky.subflunky1", None) 
>>> import mypackage.flunky.subflunky1
mypackage.__init__ executed
flunky.__init__ executed
flunky.subflunky1.__init__ executed
>>> sys.modules.get("mypackage", None) 
<module 'mypackage' from '/home/uhasker/somedir/mypackage/__init__.py'>
>>> sys.modules.get("mypackage.flunky", None) 
<module 'mypackage.flunky' from '/home/uhasker/somedir/mypackage/flunky/__init__.py'>
>>> sys.modules.get("mypackage.flunky.subflunky1", None) 
<module 'mypackage.flunky.subflunky1' from '/home/uhasker/somedir/mypackage/flunky/subflunky1/__init__.py'>
```

