# Python Language — Technical Interview Preparation Guide

> Work through each chapter sequentially. Check off topics as you complete them.

---

## Chapter 1 — [Python Basics & Setup](python/PythonBasicsAndSetup.md)

- [ ] Python history, philosophy (`import this` — The Zen of Python), design goals
- [ ] CPython, PyPy, Jython, IronPython — interpreter implementations
- [ ] Installation, `python3`, `pip`, `venv`, `pyenv` for version management
- [ ] REPL — interactive interpreter, `python -i`, IPython
- [ ] CLI tools: `python -m`, `pip install`, `pip freeze`, `pip list`
- [ ] Script execution: `if __name__ == "__main__":` guard
- [ ] Comments, docstrings (`"""..."""`), `help()`, `dir()`
- [ ] PEP 8 — style guide, naming conventions (`snake_case`, `PascalCase`, `UPPER_CASE`)
- [ ] Indentation-based syntax — blocks defined by indentation, no braces
- [ ] Dynamic typing — variables are names bound to objects, no type declarations required
- [ ] Python 2 vs Python 3 — key differences (print, division, unicode, range)

---

## Chapter 2 — [Data Types & Variables](python/DataTypesAndVariables.md)

- [ ] Everything is an object — `id()`, `type()`, `isinstance()`, `issubclass()`
- [ ] Numeric types: `int` (arbitrary precision), `float` (IEEE 754), `complex` (`3+4j`)
- [ ] Boolean: `True`, `False` — subclass of `int`
- [ ] `None` — singleton null object, identity check with `is None`
- [ ] Strings: `str` — immutable, Unicode by default (UTF-8)
- [ ] String literals: single, double, triple quotes, raw strings (`r"..."`), f-strings (`f"..."`)
- [ ] Bytes: `bytes` (immutable), `bytearray` (mutable)
- [ ] Type conversion: `int()`, `float()`, `str()`, `bool()`, `list()`, `tuple()`
- [ ] Variable assignment, multiple assignment (`a, b = 1, 2`), augmented assignment (`+=`)
- [ ] Name binding & references — variables are labels, not containers
- [ ] Mutable vs immutable types — critical distinction for function arguments & hashing
- [ ] `id()` — object identity, `is` vs `==` — identity vs equality
- [ ] Small integer caching (`-5` to `256`), string interning

---

## Chapter 3 — [Operators & Expressions](python/OperatorsAndExpressions.md)

- [ ] Arithmetic: `+`, `-`, `*`, `/` (true division), `//` (floor division), `%`, `**` (power)
- [ ] Comparison: `==`, `!=`, `<`, `>`, `<=`, `>=`
- [ ] Chained comparisons: `a < b < c` (equivalent to `a < b and b < c`)
- [ ] Logical: `and`, `or`, `not` — short-circuit evaluation, truthy/falsy values
- [ ] Bitwise: `&`, `|`, `^`, `~`, `<<`, `>>`
- [ ] Assignment: `=`, `+=`, `-=`, `*=`, `/=`, `//=`, `**=`, `%=`, `:=` (walrus operator, 3.8+)
- [ ] Membership: `in`, `not in`
- [ ] Identity: `is`, `is not`
- [ ] Ternary expression: `x if condition else y`
- [ ] Operator overloading via dunder methods (`__add__`, `__eq__`, `__lt__`, etc.)
- [ ] `divmod()`, `pow()`, `abs()`, `round()`
- [ ] Truthiness rules — `0`, `None`, empty containers are falsy; everything else is truthy

---

## Chapter 4 — [Control Flow](python/ControlFlow.md)

- [ ] `if` / `elif` / `else`
- [ ] `match` / `case` — structural pattern matching (3.10+)
  - Literal patterns, capture patterns, wildcard `_`
  - Sequence patterns, mapping patterns, class patterns
  - Guards: `case x if x > 0:`
- [ ] `for` loop — iterates over iterables (`for x in iterable:`)
- [ ] `while` loop — condition-based
- [ ] `break`, `continue`
- [ ] `for...else`, `while...else` — `else` block runs if loop completes without `break`
- [ ] `pass` — no-op placeholder
- [ ] `range()` — `range(stop)`, `range(start, stop)`, `range(start, stop, step)`
- [ ] `enumerate()`, `zip()`, `reversed()`, `sorted()`
- [ ] Nested loops and loop unpacking (`for k, v in dict.items():`)

---

## Chapter 5 — [Strings](python/Strings.md)

- [ ] String immutability — cannot modify in place
- [ ] Indexing, slicing (`s[start:stop:step]`), negative indexing
- [ ] Common methods: `split()`, `join()`, `strip()`, `replace()`, `find()`, `index()`
- [ ] Case methods: `upper()`, `lower()`, `title()`, `capitalize()`, `swapcase()`
- [ ] Check methods: `startswith()`, `endswith()`, `isdigit()`, `isalpha()`, `isalnum()`, `isspace()`
- [ ] Formatting: `%` operator, `str.format()`, f-strings (`f"{var:.2f}"`)
- [ ] f-string features (3.6+): expressions, `=` sign (3.8+, `f"{x=}"`), format specs
- [ ] `repr()` vs `str()` — developer vs user representation
- [ ] String multiplication (`"ab" * 3`), concatenation (`+`)
- [ ] `ord()`, `chr()` — character ↔ Unicode code point
- [ ] `encode()` / `decode()` — `str` ↔ `bytes` conversion
- [ ] Raw strings (`r"..."`) — no escape processing
- [ ] `textwrap`, `re` (regex) module overview
- [ ] String interning and `sys.intern()`

---

## Chapter 6 — [Lists](python/Lists.md)

- [ ] List creation: literal `[]`, `list()`, `list(iterable)`
- [ ] Indexing, slicing, slice assignment (`a[1:3] = [10, 20]`)
- [ ] Mutability — lists are mutable sequences
- [ ] Methods: `append()`, `extend()`, `insert()`, `remove()`, `pop()`, `clear()`
- [ ] Methods: `index()`, `count()`, `sort()`, `reverse()`, `copy()`
- [ ] `sorted()` vs `list.sort()` — returns new list vs in-place
- [ ] `key` parameter in sorting — `sorted(lst, key=lambda x: x[1])`
- [ ] List comprehensions: `[expr for x in iterable if cond]`
- [ ] Nested list comprehensions
- [ ] Shallow copy vs deep copy — `copy()`, `list()`, `[:]`, `copy.deepcopy()`
- [ ] List as stack (`append` / `pop`) and queue (use `collections.deque` instead)
- [ ] List multiplication gotcha: `[[]] * 3` — all elements share same reference
- [ ] `*` unpacking: `first, *rest = [1, 2, 3, 4]`
- [ ] Time complexity: `append` O(1) amortized, `insert(0)` O(n), `in` O(n)

---

## Chapter 7 — [Tuples](python/Tuples.md)

- [ ] Tuple creation: `()`, `(1,)` (single-element needs trailing comma), `tuple()`
- [ ] Immutability — cannot add, remove, or reassign elements
- [ ] Tuple packing and unpacking: `a, b, c = (1, 2, 3)`
- [ ] `*` unpacking in tuples: `first, *middle, last = (1, 2, 3, 4, 5)`
- [ ] Named tuples: `collections.namedtuple("Point", ["x", "y"])`
- [ ] `typing.NamedTuple` — class-based syntax with type hints
- [ ] Tuples as dictionary keys (hashable if all elements are hashable)
- [ ] Tuples vs lists — when to use which
- [ ] `count()`, `index()` — only two methods
- [ ] Tuple comparison — lexicographic order
- [ ] Immutable but can contain mutable objects: `([1, 2], [3, 4])`

---

## Chapter 8 — [Sets](python/Sets.md)

- [ ] Set creation: `{1, 2, 3}`, `set()` (empty set — `{}` creates dict)
- [ ] Unordered, unique elements, elements must be hashable
- [ ] Methods: `add()`, `remove()`, `discard()`, `pop()`, `clear()`
- [ ] Set operations: `union()` / `|`, `intersection()` / `&`, `difference()` / `-`, `symmetric_difference()` / `^`
- [ ] Subset / superset: `issubset()` / `<=`, `issuperset()` / `>=`
- [ ] `frozenset` — immutable set, hashable, can be used as dict key or set element
- [ ] Set comprehensions: `{expr for x in iterable if cond}`
- [ ] Time complexity: `add` O(1), `in` O(1), `remove` O(1)
- [ ] Use cases: membership testing, deduplication, mathematical set operations

---

## Chapter 9 — [Dictionaries](python/Dictionaries.md)

- [ ] Dict creation: `{}`, `dict()`, `dict(a=1, b=2)`, `dict(zip(keys, vals))`
- [ ] Access: `d[key]` (KeyError if missing), `d.get(key, default)`
- [ ] Methods: `keys()`, `values()`, `items()` — return view objects
- [ ] Methods: `update()`, `pop()`, `popitem()`, `setdefault()`, `clear()`, `copy()`
- [ ] `del d[key]` — delete entry
- [ ] Dictionary comprehensions: `{k: v for k, v in iterable}`
- [ ] `collections.defaultdict` — auto-creates missing keys with factory
- [ ] `collections.OrderedDict` — preserves insertion order (dict does too since 3.7+)
- [ ] `collections.Counter` — count hashable objects, `most_common()`, arithmetic ops
- [ ] `collections.ChainMap` — search multiple dicts as one
- [ ] Merge operators (3.9+): `d1 | d2`, `d1 |= d2`
- [ ] Dict ordering — insertion order guaranteed since Python 3.7
- [ ] `**` unpacking: `{**d1, **d2}`
- [ ] Keys must be hashable — `__hash__` and `__eq__` requirements
- [ ] Time complexity: `get` O(1), `set` O(1), `in` O(1), `del` O(1) average

---

## Chapter 10 — [Functions](python/Functions.md)

- [ ] Function definition: `def func_name(params):`
- [ ] `return` statement — returns `None` if omitted
- [ ] Positional arguments, keyword arguments
- [ ] Default parameter values — **mutable default argument trap** (`def f(lst=[]):`)
- [ ] `*args` — variable positional arguments (tuple)
- [ ] `**kwargs` — variable keyword arguments (dict)
- [ ] Argument order: `def f(pos, /, pos_or_kw, *, kw_only):`
- [ ] Positional-only (`/`) and keyword-only (`*`) parameters (3.8+)
- [ ] First-class functions — functions as objects, passed as arguments, returned
- [ ] `lambda` expressions: `lambda x, y: x + y`
- [ ] Closures — nested functions capturing enclosing scope variables
- [ ] `nonlocal` keyword — modify enclosing scope variable
- [ ] `global` keyword — modify module-level variable
- [ ] Recursion, recursion limit (`sys.getrecursionlimit()`, `sys.setrecursionlimit()`)
- [ ] `*` and `**` unpacking in function calls: `f(*args, **kwargs)`
- [ ] Type hints / annotations: `def f(x: int) -> str:`
- [ ] Docstrings — `"""..."""`, `__doc__` attribute

---

## Chapter 11 — [Scope & Namespaces (LEGB Rule)](python/ScopeAndNamespaces.md)

- [ ] LEGB rule: Local → Enclosing → Global → Built-in
- [ ] Local scope — variables defined inside a function
- [ ] Enclosing scope — variables in outer function (closures)
- [ ] Global scope — module-level variables
- [ ] Built-in scope — `print`, `len`, `range`, etc.
- [ ] `global` statement — declare variable as global inside function
- [ ] `nonlocal` statement — declare variable from enclosing scope
- [ ] `globals()` and `locals()` — introspect namespaces
- [ ] Name resolution order — how Python looks up names
- [ ] Variable shadowing — local variable hides outer variable
- [ ] `UnboundLocalError` — referencing before assignment in local scope

---

## Chapter 12 — [Decorators](python/Decorators.md)

- [ ] Decorator syntax: `@decorator` — syntactic sugar for `func = decorator(func)`
- [ ] Function decorators — wrapping functions to add behavior
- [ ] `functools.wraps` — preserve original function metadata
- [ ] Decorators with arguments: `@decorator(arg)` — decorator factory pattern
- [ ] Stacking decorators — execution order (bottom-up application, top-down execution)
- [ ] Class decorators — decorating classes
- [ ] Common built-in decorators: `@staticmethod`, `@classmethod`, `@property`
- [ ] `@functools.lru_cache` — memoization decorator
- [ ] `@functools.cache` (3.9+) — unbounded cache
- [ ] `@functools.singledispatch` — single-dispatch generic functions
- [ ] `@dataclasses.dataclass` — auto-generate `__init__`, `__repr__`, etc.
- [ ] Real-world use cases: logging, timing, authentication, retry logic, validation

---

## Chapter 13 — [Generators & Iterators](python/GeneratorsAndIterators.md)

- [ ] Iterator protocol: `__iter__()` returns iterator, `__next__()` returns next element
- [ ] `StopIteration` exception — signals end of iteration
- [ ] Building custom iterators — class with `__iter__` and `__next__`
- [ ] `iter()` function — `iter(object)`, `iter(callable, sentinel)`
- [ ] Generator functions — `yield` keyword, lazy evaluation
- [ ] Generator state — suspended execution, resumed on `next()`
- [ ] `yield from` — delegate to sub-generator (3.3+)
- [ ] Generator expressions: `(expr for x in iterable if cond)`
- [ ] Generator vs list comprehension — memory efficiency, single-use
- [ ] `send()` — send value into generator, coroutine-like behavior
- [ ] `throw()` — throw exception into generator
- [ ] `close()` — close generator, triggers `GeneratorExit`
- [ ] Infinite generators — `itertools.count()`, custom infinite sequences
- [ ] `itertools` module:
  - `chain()`, `islice()`, `cycle()`, `repeat()`
  - `combinations()`, `permutations()`, `product()`
  - `groupby()`, `accumulate()`, `starmap()`
  - `tee()`, `zip_longest()`, `filterfalse()`
- [ ] Memory benefits — generators process one item at a time, no full list in memory

---

## Chapter 14 — [Comprehensions](python/Comprehensions.md)

- [ ] List comprehensions: `[expr for x in iterable if cond]`
- [ ] Dict comprehensions: `{k: v for k, v in iterable if cond}`
- [ ] Set comprehensions: `{expr for x in iterable if cond}`
- [ ] Generator expressions: `(expr for x in iterable if cond)`
- [ ] Nested comprehensions — multiple `for` clauses
- [ ] Conditional expressions in comprehensions — `if`/`else` in expression vs filter
- [ ] Walrus operator in comprehensions (3.8+): `[y := f(x) for x in lst if y > 0]`
- [ ] When to use comprehensions vs loops — readability threshold
- [ ] Variable leakage — comprehension variables don't leak in Python 3 (they do in Python 2)
- [ ] Performance — comprehensions are generally faster than equivalent `for` loops

---

## Chapter 15 — [Object-Oriented Programming](python/OOP.md)

- [ ] Class definition: `class MyClass:`, `class MyClass(Base):`
- [ ] `__init__` — constructor / initializer
- [ ] `self` — explicit reference to instance
- [ ] Instance attributes vs class attributes
- [ ] Methods: instance methods, `@classmethod` (receives `cls`), `@staticmethod`
- [ ] `@property` — getter, setter, deleter
- [ ] Inheritance — single inheritance, `super()` for parent method calls
- [ ] Multiple inheritance — MRO (Method Resolution Order), C3 linearization
- [ ] `super()` — cooperative multiple inheritance
- [ ] `isinstance()`, `issubclass()` — type checking
- [ ] Abstract base classes: `abc.ABC`, `@abc.abstractmethod`
- [ ] Encapsulation conventions: `_private`, `__name_mangling`
- [ ] Name mangling — `__attr` becomes `_ClassName__attr`
- [ ] `__slots__` — restrict attributes, save memory, faster access
- [ ] Mixins — small, focused classes for reusable behavior
- [ ] Composition vs inheritance — "has-a" vs "is-a"
- [ ] Dataclasses (3.7+): `@dataclass`, `field()`, frozen, `__post_init__`
- [ ] `__repr__`, `__str__`, `__eq__`, `__hash__` — common dunder methods

---

## Chapter 16 — [Dunder (Magic) Methods](python/DunderMethods.md)

### Object Lifecycle
- [ ] `__new__` — object creation (before `__init__`)
- [ ] `__init__` — object initialization
- [ ] `__del__` — destructor (called when refcount drops to 0, not guaranteed)

### String Representation
- [ ] `__repr__` — unambiguous representation (for developers)
- [ ] `__str__` — readable representation (for users)
- [ ] `__format__` — custom format specs (`f"{obj:spec}"`)
- [ ] `__bytes__` — `bytes(obj)` conversion

### Comparison
- [ ] `__eq__`, `__ne__`, `__lt__`, `__le__`, `__gt__`, `__ge__`
- [ ] `@functools.total_ordering` — generate missing comparisons from `__eq__` + one other

### Arithmetic
- [ ] `__add__`, `__sub__`, `__mul__`, `__truediv__`, `__floordiv__`, `__mod__`, `__pow__`
- [ ] Reflected: `__radd__`, `__rsub__`, etc. — called when left operand doesn't support operation
- [ ] In-place: `__iadd__`, `__isub__`, etc.
- [ ] Unary: `__neg__`, `__pos__`, `__abs__`, `__invert__`

### Container Protocol
- [ ] `__len__` — `len(obj)`
- [ ] `__getitem__` — `obj[key]`
- [ ] `__setitem__` — `obj[key] = value`
- [ ] `__delitem__` — `del obj[key]`
- [ ] `__contains__` — `item in obj`
- [ ] `__iter__` — `iter(obj)`
- [ ] `__next__` — `next(obj)`
- [ ] `__reversed__` — `reversed(obj)`

### Attribute Access
- [ ] `__getattr__` — called when attribute not found
- [ ] `__getattribute__` — called on every attribute access
- [ ] `__setattr__` — called on every attribute assignment
- [ ] `__delattr__` — called on attribute deletion

### Callable & Context Manager
- [ ] `__call__` — make object callable: `obj()`
- [ ] `__enter__`, `__exit__` — context manager protocol (`with` statement)

### Hashing
- [ ] `__hash__` — `hash(obj)`, must be consistent with `__eq__`
- [ ] If `__eq__` is defined without `__hash__`, object becomes unhashable

---

## Chapter 17 — [Error & Exception Handling](python/ErrorHandling.md)

- [ ] Exception hierarchy: `BaseException` → `Exception` → specific exceptions
- [ ] `try` / `except` / `else` / `finally`
- [ ] Catching specific exceptions: `except ValueError as e:`
- [ ] Catching multiple: `except (TypeError, ValueError):`
- [ ] `raise` — raise exceptions, `raise ValueError("msg")`
- [ ] `raise ... from ...` — exception chaining (explicit cause)
- [ ] Custom exceptions — inherit from `Exception`
- [ ] `finally` — always executes (cleanup code)
- [ ] `else` — runs only if no exception was raised
- [ ] Built-in exceptions: `ValueError`, `TypeError`, `KeyError`, `IndexError`, `AttributeError`, `FileNotFoundError`, `IOError`, `RuntimeError`, `StopIteration`, `NotImplementedError`
- [ ] `assert` statement — assertions for debugging, disabled with `-O` flag
- [ ] Exception groups & `except*` (3.11+) — `ExceptionGroup`, `except* ValueError:`
- [ ] `add_note()` (3.11+) — add context notes to exceptions
- [ ] EAFP vs LBYL — "Easier to Ask Forgiveness than Permission" (Python idiom)
- [ ] Bare `except:` — catches everything including `SystemExit`, `KeyboardInterrupt` (avoid)
- [ ] Context-aware exception handling — logging, retry patterns

---

## Chapter 18 — [File I/O](python/FileIO.md)

- [ ] `open()` — `open(path, mode, encoding)`, file modes: `r`, `w`, `a`, `x`, `b`, `t`, `+`
- [ ] `with` statement — context manager for automatic file closing
- [ ] Reading: `read()`, `readline()`, `readlines()`, iterating over file object
- [ ] Writing: `write()`, `writelines()`
- [ ] Binary mode: `'rb'`, `'wb'` — reading/writing bytes
- [ ] `seek()`, `tell()` — file position manipulation
- [ ] `os` module: `os.path.exists()`, `os.makedirs()`, `os.listdir()`, `os.rename()`, `os.remove()`
- [ ] `pathlib` module (3.4+): `Path` object — `Path.read_text()`, `Path.write_text()`, `Path.glob()`, `Path.mkdir()`, `/` operator for path joining
- [ ] `shutil` — high-level file operations: `copy()`, `copytree()`, `rmtree()`, `move()`
- [ ] `tempfile` — temporary files and directories: `NamedTemporaryFile`, `TemporaryDirectory`
- [ ] `json` module: `json.dump()`, `json.load()`, `json.dumps()`, `json.loads()`
- [ ] `csv` module: `csv.reader()`, `csv.writer()`, `csv.DictReader()`, `csv.DictWriter()`
- [ ] `pickle` module — serialize/deserialize Python objects (security warning)
- [ ] File encoding — `encoding='utf-8'`, `errors='strict'`/`'ignore'`/`'replace'`

---

## Chapter 19 — [Modules & Packages](python/ModulesAndPackages.md)

- [ ] Module — single `.py` file, imported with `import module`
- [ ] `import`, `from ... import ...`, `import ... as ...`
- [ ] `__name__` — `"__main__"` when run directly, module name when imported
- [ ] `__all__` — controls `from module import *` behavior
- [ ] Package — directory with `__init__.py`
- [ ] `__init__.py` — package initialization, can be empty or contain code
- [ ] Relative imports: `from . import sibling`, `from .. import parent_module`
- [ ] Absolute vs relative imports — best practices
- [ ] `sys.path` — module search path, how Python finds modules
- [ ] `sys.modules` — cache of loaded modules
- [ ] Circular imports — causes and solutions
- [ ] `importlib` — dynamic imports: `importlib.import_module("name")`
- [ ] `pip` — package installer: `install`, `uninstall`, `freeze`, `requirements.txt`
- [ ] Virtual environments: `venv`, `virtualenv`, `pipenv`, `poetry`, `uv`
- [ ] `setup.py`, `setup.cfg`, `pyproject.toml` — packaging and distribution
- [ ] Namespace packages — packages without `__init__.py` (3.3+)

---

## Chapter 20 — [Type Hints & Static Typing](python/TypeHintsAndStaticTyping.md)

- [ ] Basic annotations: `x: int = 5`, `def f(x: int) -> str:`
- [ ] `typing` module: `List`, `Dict`, `Tuple`, `Set`, `Optional`, `Union`
- [ ] Built-in generics (3.9+): `list[int]`, `dict[str, int]`, `tuple[int, ...]`
- [ ] `Optional[X]` — equivalent to `Union[X, None]`, `X | None` (3.10+)
- [ ] Union types: `Union[int, str]`, `int | str` (3.10+)
- [ ] `Any` — opt out of type checking
- [ ] `TypeVar` — generic type variables: `T = TypeVar("T")`
- [ ] `Generic` — defining generic classes
- [ ] `Protocol` — structural subtyping (duck typing with types)
- [ ] `TypeAlias` (3.10+) — explicit type alias declaration
- [ ] `TypeGuard` (3.10+) — narrowing types in conditional checks
- [ ] `Literal` — restrict to specific values: `Literal["red", "blue"]`
- [ ] `Final` — constants that cannot be reassigned
- [ ] `ClassVar` — class-level variable annotation
- [ ] `Callable` — callable type: `Callable[[int, str], bool]`
- [ ] `TypedDict` — typed dictionaries with specific keys
- [ ] `ParamSpec`, `Concatenate` (3.10+) — for decorator type hints
- [ ] `Self` type (3.11+) — reference to current class in methods
- [ ] `type` statement (3.12+) — type alias syntax: `type Point = tuple[int, int]`
- [ ] `mypy` — static type checker, configuration, strict mode
- [ ] Runtime behavior — annotations are **not enforced** at runtime by default

---

## Chapter 21 — [Concurrency & Parallelism](python/ConcurrencyAndParallelism.md)

### Threading
- [ ] `threading` module: `Thread`, `Lock`, `RLock`, `Semaphore`, `Event`, `Condition`, `Barrier`
- [ ] GIL (Global Interpreter Lock) — only one thread executes Python bytecode at a time
- [ ] GIL implications — threads useful for I/O-bound, not CPU-bound tasks
- [ ] Thread safety — race conditions, `Lock` for shared state
- [ ] `threading.local()` — thread-local storage
- [ ] Daemon threads — `daemon=True`, don't prevent program exit
- [ ] Free-threaded Python (3.13+) — experimental no-GIL mode

### Multiprocessing
- [ ] `multiprocessing` module: `Process`, `Pool`, `Queue`, `Pipe`, `Manager`
- [ ] True parallelism — separate processes, separate GIL per process
- [ ] `Pool.map()`, `Pool.starmap()`, `Pool.apply_async()`
- [ ] Shared state: `Value`, `Array`, `Manager` — inter-process communication
- [ ] `ProcessPoolExecutor` — `concurrent.futures` interface

### Async / Await
- [ ] `asyncio` — event loop, coroutines, tasks
- [ ] `async def` — coroutine function
- [ ] `await` — suspend coroutine until awaitable completes
- [ ] `asyncio.run()` — entry point for async code
- [ ] `asyncio.create_task()` — schedule coroutine concurrently
- [ ] `asyncio.gather()` — run multiple coroutines concurrently
- [ ] `asyncio.wait()`, `asyncio.as_completed()`
- [ ] `asyncio.Queue` — async producer-consumer
- [ ] `asyncio.Semaphore`, `asyncio.Lock` — async synchronization
- [ ] `asyncio.sleep()` vs `time.sleep()` — non-blocking vs blocking
- [ ] `async for`, `async with` — async iteration and context managers
- [ ] `aiohttp`, `httpx` — async HTTP clients

### concurrent.futures
- [ ] `ThreadPoolExecutor`, `ProcessPoolExecutor`
- [ ] `executor.submit()` → `Future`, `future.result()`
- [ ] `executor.map()` — parallel map
- [ ] `as_completed()` — iterate results as they finish

---

## Chapter 22 — [Context Managers](python/ContextManagers.md)

- [ ] `with` statement — resource management with automatic cleanup
- [ ] Context manager protocol: `__enter__()`, `__exit__(exc_type, exc_val, exc_tb)`
- [ ] `__exit__` return value — return `True` to suppress exception
- [ ] `contextlib.contextmanager` — decorator to create context managers from generators
- [ ] `contextlib.suppress(*exceptions)` — suppress specific exceptions
- [ ] `contextlib.redirect_stdout`, `contextlib.redirect_stderr`
- [ ] `contextlib.ExitStack` — manage dynamic number of context managers
- [ ] `contextlib.asynccontextmanager` — async context managers
- [ ] Nested `with` statements — `with open(a) as f1, open(b) as f2:`
- [ ] Parenthesized context managers (3.10+): `with (open(a) as f1, open(b) as f2):`
- [ ] Use cases: file handling, database connections, locks, temporary state changes

---

## Chapter 23 — [Regular Expressions](python/RegularExpressions.md)

- [ ] `re` module: `re.search()`, `re.match()`, `re.fullmatch()`, `re.findall()`, `re.finditer()`
- [ ] `re.sub()`, `re.subn()` — substitution
- [ ] `re.split()` — split by pattern
- [ ] `re.compile()` — pre-compile patterns for reuse
- [ ] Match objects: `group()`, `groups()`, `groupdict()`, `start()`, `end()`, `span()`
- [ ] Metacharacters: `.`, `^`, `$`, `*`, `+`, `?`, `{}`, `[]`, `|`, `()`
- [ ] Character classes: `\d`, `\w`, `\s`, `\b` and negations `\D`, `\W`, `\S`, `\B`
- [ ] Quantifiers: `*` (0+), `+` (1+), `?` (0 or 1), `{n}`, `{n,m}`
- [ ] Greedy vs non-greedy: `*?`, `+?`, `??`
- [ ] Groups: capturing `()`, named `(?P<name>...)`, non-capturing `(?:...)`
- [ ] Lookahead `(?=...)`, lookbehind `(?<=...)`, negative versions
- [ ] Flags: `re.IGNORECASE`, `re.MULTILINE`, `re.DOTALL`, `re.VERBOSE`
- [ ] Raw strings (`r"..."`) — preferred for regex patterns

---

## Chapter 24 — [Functional Programming](python/FunctionalProgramming.md)

- [ ] First-class functions — functions as objects
- [ ] `lambda` — anonymous functions
- [ ] `map()` — apply function to iterable
- [ ] `filter()` — filter elements by predicate
- [ ] `reduce()` — `functools.reduce()`, fold/accumulate
- [ ] `functools.partial()` — partial function application
- [ ] `functools.lru_cache` / `functools.cache` — memoization
- [ ] `functools.wraps` — preserve wrapped function metadata
- [ ] `functools.total_ordering` — auto-generate comparison methods
- [ ] `operator` module: `operator.add`, `operator.itemgetter`, `operator.attrgetter`
- [ ] `itertools` — combinatoric, infinite, and terminating iterators
- [ ] Closures and higher-order functions
- [ ] Immutability patterns — `frozenset`, `tuple`, `types.MappingProxyType`
- [ ] Pure functions — no side effects, deterministic output
- [ ] Function composition — manual chaining, no built-in compose

---

## Chapter 25 — [Memory Management & Internals](python/MemoryManagementAndInternals.md)

- [ ] Reference counting — primary garbage collection mechanism
- [ ] `sys.getrefcount()` — check reference count
- [ ] Cyclic garbage collector — detects reference cycles (`gc` module)
- [ ] `gc.collect()`, `gc.disable()`, `gc.get_objects()`
- [ ] Generational GC — 3 generations (gen0, gen1, gen2), promotion thresholds
- [ ] Memory allocation — CPython memory allocator, `pymalloc`, object-specific allocators
- [ ] Object interning — small integers (`-5` to `256`), string interning
- [ ] `__slots__` — reduce memory by avoiding `__dict__`
- [ ] `sys.getsizeof()` — size of object in bytes (shallow)
- [ ] `weakref` — weak references, `weakref.ref()`, `WeakValueDictionary`
- [ ] Copy semantics — assignment creates reference, `copy.copy()` (shallow), `copy.deepcopy()` (deep)
- [ ] CPython bytecode — `dis` module, `dis.dis()`, understanding bytecode
- [ ] Python object model — `PyObject`, type object, refcount, value
- [ ] Memory leaks — common causes (circular refs, global caches, `__del__` issues)

---

## Chapter 26 — [Metaclasses & Descriptors](python/MetaclassesAndDescriptors.md)

### Descriptors
- [ ] Descriptor protocol: `__get__`, `__set__`, `__delete__`, `__set_name__`
- [ ] Data descriptor vs non-data descriptor
- [ ] `property` is a descriptor — how `@property` works internally
- [ ] Custom descriptors — validation, lazy computation, type enforcement

### Metaclasses
- [ ] What is a metaclass — "class of a class", `type` is the default metaclass
- [ ] `type()` — dynamic class creation: `type("MyClass", (Base,), {"attr": val})`
- [ ] Custom metaclasses: `class Meta(type):`, `__new__`, `__init__`, `__call__`
- [ ] Metaclass `__new__` vs `__init__` — when each is called
- [ ] `__init_subclass__` (3.6+) — simpler alternative to metaclasses
- [ ] `__class_getitem__` (3.7+) — enable `MyClass[int]` syntax
- [ ] Class creation process — `__prepare__`, `__new__`, `__init__`
- [ ] ABCMeta — `abc.ABCMeta` as metaclass, `abc.ABC` convenience
- [ ] Real-world uses — ORMs, API frameworks, registration patterns, singleton

---

## Chapter 27 — [Testing](python/Testing.md)

- [ ] `unittest` module:
  - `TestCase`, `setUp()`, `tearDown()`, `setUpClass()`, `tearDownClass()`
  - Assertions: `assertEqual`, `assertTrue`, `assertRaises`, `assertIn`, `assertIsNone`
  - Test discovery: `python -m unittest discover`
  - `unittest.mock` — `Mock`, `MagicMock`, `patch`, `patch.object`
- [ ] `pytest` framework:
  - Test functions: `def test_something():`, `assert` statement
  - Fixtures: `@pytest.fixture`, scope (`function`, `class`, `module`, `session`)
  - Parametrize: `@pytest.mark.parametrize("input,expected", [...])`
  - Markers: `@pytest.mark.skip`, `@pytest.mark.xfail`, custom markers
  - `conftest.py` — shared fixtures and hooks
  - `pytest.raises` — assert exceptions
  - Plugins: `pytest-cov`, `pytest-mock`, `pytest-asyncio`, `pytest-xdist`
- [ ] `doctest` — tests embedded in docstrings: `python -m doctest module.py`
- [ ] Mocking best practices — mock at the import location, `spec=True`
- [ ] Test coverage: `coverage.py`, `pytest-cov`
- [ ] `hypothesis` — property-based testing

---

## Chapter 28 — [Standard Library Highlights](python/StandardLibraryHighlights.md)

- [ ] `collections` — `deque`, `defaultdict`, `Counter`, `OrderedDict`, `ChainMap`, `namedtuple`
- [ ] `itertools` — combinatoric and infinite iterators
- [ ] `functools` — `lru_cache`, `partial`, `reduce`, `wraps`, `total_ordering`
- [ ] `operator` — function versions of operators
- [ ] `os` — OS interface: environment, paths, process management
- [ ] `sys` — interpreter info: `argv`, `path`, `stdin/stdout/stderr`, `exit()`
- [ ] `pathlib` — object-oriented filesystem paths
- [ ] `datetime` — `date`, `time`, `datetime`, `timedelta`, `timezone`
- [ ] `time` — `time()`, `sleep()`, `perf_counter()`, `monotonic()`
- [ ] `math` — `ceil`, `floor`, `sqrt`, `log`, `factorial`, `gcd`, `inf`, `nan`
- [ ] `random` — `random()`, `randint()`, `choice()`, `shuffle()`, `sample()`
- [ ] `secrets` — cryptographic randomness: `token_hex()`, `token_urlsafe()`
- [ ] `hashlib` — `md5`, `sha256`, `sha512`, `pbkdf2_hmac`
- [ ] `logging` — `getLogger()`, handlers, formatters, log levels
- [ ] `argparse` — command-line argument parsing
- [ ] `subprocess` — `run()`, `Popen`, capturing output, piping
- [ ] `socket` — low-level networking
- [ ] `http.server`, `urllib`, `http.client` — HTTP tools
- [ ] `sqlite3` — built-in SQLite database interface
- [ ] `dataclasses` — `@dataclass`, `field()`, `asdict()`, `astuple()`
- [ ] `enum` — `Enum`, `IntEnum`, `Flag`, `auto()`, `StrEnum` (3.11+)
- [ ] `abc` — abstract base classes
- [ ] `typing` — type hints
- [ ] `struct` — pack/unpack binary data
- [ ] `heapq` — heap queue: `heappush`, `heappop`, `nlargest`, `nsmallest`
- [ ] `bisect` — binary search: `bisect_left`, `bisect_right`, `insort`
- [ ] `array` — typed arrays (more compact than lists for numeric data)
- [ ] `queue` — `Queue`, `LifoQueue`, `PriorityQueue` (thread-safe)
- [ ] `textwrap` — wrapping and filling text
- [ ] `string` — `Template`, `ascii_letters`, `digits`, constants
- [ ] `pprint` — pretty-print data structures

---

## Chapter 29 — [Networking & Web](python/NetworkingAndWeb.md)

- [ ] `socket` — TCP/UDP sockets: `socket()`, `bind()`, `listen()`, `accept()`, `connect()`
- [ ] `http.server` — simple HTTP server: `HTTPServer`, `SimpleHTTPRequestHandler`
- [ ] `urllib` — `urllib.request.urlopen()`, `urllib.parse`
- [ ] `requests` library — `get()`, `post()`, `put()`, `delete()`, sessions, auth
- [ ] `httpx` — async-capable HTTP client
- [ ] `aiohttp` — async HTTP client and server
- [ ] `flask` — micro web framework overview
- [ ] `fastapi` — async web framework with type hints, automatic docs
- [ ] `django` — full-stack web framework overview
- [ ] REST API design patterns — endpoints, status codes, serialization
- [ ] `websockets` / `websocket` — WebSocket communication
- [ ] `xmlrpc`, `json-rpc` — RPC protocols
- [ ] `ssl` — TLS/SSL wrappers for sockets

---

## Chapter 30 — [Database Access](python/DatabaseAccess.md)

- [ ] DB-API 2.0 (PEP 249) — standard database interface
- [ ] `sqlite3` — built-in SQLite: `connect()`, `cursor()`, `execute()`, `fetchall()`
- [ ] Parameterized queries — `?` placeholders, prevent SQL injection
- [ ] Context managers for connections and cursors
- [ ] `psycopg2` / `psycopg3` — PostgreSQL driver
- [ ] `mysql-connector-python` / `PyMySQL` — MySQL drivers
- [ ] `SQLAlchemy` — ORM and Core: engine, session, models, queries
- [ ] `SQLAlchemy ORM` — declarative base, relationships, lazy loading
- [ ] Connection pooling — `pool_size`, `max_overflow`
- [ ] Transactions — `commit()`, `rollback()`, autocommit
- [ ] Migrations — `alembic` for SQLAlchemy, `django.db.migrations`
- [ ] `pymongo` — MongoDB driver overview
- [ ] `redis-py` — Redis client overview
- [ ] Async database drivers — `asyncpg`, `aiosqlite`, `databases`

---

## Chapter 31 — [Design Patterns in Python](python/DesignPatterns.md)

- [ ] **Singleton** — module-level instance, `__new__` override, metaclass approach
- [ ] **Factory** — factory function, `@classmethod` alternative constructors
- [ ] **Builder** — method chaining, step-by-step construction
- [ ] **Strategy** — swap behavior via callable / class instances
- [ ] **Observer** — event-driven callbacks, `signal` libraries
- [ ] **Decorator** — function/class wrapping (overlaps with Python decorators)
- [ ] **Adapter** — convert one interface to another
- [ ] **Proxy** — lazy loading, access control, `__getattr__` forwarding
- [ ] **Iterator** — `__iter__` / `__next__` protocol
- [ ] **Template Method** — abstract base class with concrete steps
- [ ] **Command** — encapsulate operations as objects
- [ ] **State** — object behavior changes based on internal state
- [ ] **Dependency Injection** — pass dependencies via constructor/parameters
- [ ] **Repository** — abstract data access layer
- [ ] **Context Manager** — RAII pattern via `with` statement

---

## Chapter 32 — [Performance & Profiling](python/PerformanceAndProfiling.md)

- [ ] `timeit` — micro-benchmarking: `timeit.timeit()`, `python -m timeit`
- [ ] `cProfile` / `profile` — function-level profiling: `cProfile.run()`, `python -m cProfile`
- [ ] `pstats` — analyze profiling results
- [ ] `line_profiler` — line-by-line profiling (`@profile` decorator)
- [ ] `memory_profiler` — memory usage profiling
- [ ] `tracemalloc` — trace memory allocations (built-in)
- [ ] Common optimizations:
  - Use built-in functions and data structures
  - List comprehensions over loops
  - `set` for membership testing
  - `join()` for string concatenation
  - Avoid global variable lookups in tight loops
  - `__slots__` for memory-bound classes
  - Generator expressions for large datasets
- [ ] `Cython` — compile Python to C extension
- [ ] `ctypes`, `cffi` — calling C libraries
- [ ] `numpy` — vectorized operations for numeric computing
- [ ] `PyPy` — JIT-compiled Python interpreter
- [ ] `dis` module — disassemble bytecode for analysis
- [ ] Big-O awareness — choosing right data structure for operations

---

## Chapter 33 — [Packaging & Distribution](python/PackagingAndDistribution.md)

- [ ] `pyproject.toml` — modern project configuration (PEP 518, PEP 621)
- [ ] `setup.py`, `setup.cfg` — legacy packaging
- [ ] Build tools: `setuptools`, `flit`, `hatch`, `poetry`, `pdm`
- [ ] `pip` — install packages: `pip install`, `pip install -e .` (editable)
- [ ] `requirements.txt` — dependency pinning
- [ ] `pip-tools` — `pip-compile` for deterministic builds
- [ ] `wheel` vs `sdist` — binary vs source distribution
- [ ] PyPI — uploading with `twine`, `build`
- [ ] Virtual environments: `venv`, `virtualenv`, `conda`
- [ ] `uv` — fast Python package installer (Rust-based)
- [ ] Dependency management: `poetry`, `pipenv`, `pdm`
- [ ] Lock files — reproducible installs
- [ ] `__version__` — versioning strategies, `importlib.metadata`

---

## Chapter 34 — [Common Interview Gotchas & Tricky Questions](python/InterviewGotchas.md)

- [ ] **Mutable default arguments** — `def f(lst=[]):` shares list across calls; use `None` sentinel
- [ ] **`is` vs `==`** — identity vs equality; small integer caching (`-5` to `256`)
- [ ] **Late binding closures** — `lambda` in loop captures variable by reference, not value
- [ ] **List multiplication trap** — `[[]] * 3` creates 3 references to same inner list
- [ ] **String immutability** — concatenation in loop is O(n²); use `"".join()`
- [ ] **Dictionary key ordering** — guaranteed insertion order since Python 3.7
- [ ] **`copy()` vs `deepcopy()`** — shallow copy doesn't clone nested objects
- [ ] **GIL** — threading doesn't speed up CPU-bound code; use `multiprocessing`
- [ ] **`__init__` vs `__new__`** — `__new__` creates, `__init__` initializes
- [ ] **Tuple with one element** — `(1)` is int, `(1,)` is tuple
- [ ] **`except Exception` vs bare `except:`** — bare catches `SystemExit`, `KeyboardInterrupt`
- [ ] **Float precision** — `0.1 + 0.2 != 0.3`; use `decimal.Decimal` for precision
- [ ] **Variable scope in loops** — loop variable persists after loop ends
- [ ] **Chained assignment** — `a = b = []` both reference same list
- [ ] **`__all__` and wildcard imports** — controls `from module import *`
- [ ] **Circular imports** — causes `ImportError`; restructure or use lazy imports
- [ ] **Class vs instance attributes** — mutable class attribute shared across instances
- [ ] **`else` on loops** — `for...else` and `while...else` — `else` runs if no `break`
- [ ] **Unpacking gotchas** — `a, b = b, a` works (simultaneous swap), but `a, b = b, a+b` evaluates RHS first

---

## Chapter 35 — [Python Versions & Key Features](python/PythonVersions.md)

| Version | Key Features |
|---------|-------------|
| Python 3.6 | f-strings, `__init_subclass__`, `secrets` module, variable annotations |
| Python 3.7 | `dataclasses`, `breakpoint()`, dict insertion order guaranteed, `async`/`await` reserved |
| Python 3.8 | Walrus operator (`:=`), positional-only params (`/`), `f"{x=}"` debugging |
| Python 3.9 | Dict merge (`\|`), `list[int]` built-in generics, `str.removeprefix/suffix` |
| Python 3.10 | `match`/`case` pattern matching, `\|` union types in annotations, `ParamSpec` |
| Python 3.11 | Exception groups, `ExceptionGroup`, `tomllib`, `Self` type, faster CPython |
| Python 3.12 | `type` statement, `f-string` improvements, per-interpreter GIL, `@override` |
| Python 3.13 | Free-threaded mode (no-GIL experimental), improved error messages, JIT compiler (experimental) |

---

*Last updated: March 10, 2026*

---

## Study Checklist

- [ ] Chapter 1: [Python Basics & Setup](python/PythonBasicsAndSetup.md)
- [ ] Chapter 2: [Data Types & Variables](python/DataTypesAndVariables.md)
- [ ] Chapter 3: [Operators & Expressions](python/OperatorsAndExpressions.md)
- [ ] Chapter 4: [Control Flow](python/ControlFlow.md)
- [ ] Chapter 5: [Strings](python/Strings.md)
- [ ] Chapter 6: [Lists](python/Lists.md)
- [ ] Chapter 7: [Tuples](python/Tuples.md)
- [ ] Chapter 8: [Sets](python/Sets.md)
- [ ] Chapter 9: [Dictionaries](python/Dictionaries.md)
- [ ] Chapter 10: [Functions](python/Functions.md)
- [ ] Chapter 11: [Scope & Namespaces (LEGB Rule)](python/ScopeAndNamespaces.md)
- [ ] Chapter 12: [Decorators](python/Decorators.md)
- [ ] Chapter 13: [Generators & Iterators](python/GeneratorsAndIterators.md)
- [ ] Chapter 14: [Comprehensions](python/Comprehensions.md)
- [ ] Chapter 15: [Object-Oriented Programming](python/OOP.md)
- [ ] Chapter 16: [Dunder (Magic) Methods](python/DunderMethods.md)
- [ ] Chapter 17: [Error & Exception Handling](python/ErrorHandling.md)
- [ ] Chapter 18: [File I/O](python/FileIO.md)
- [ ] Chapter 19: [Modules & Packages](python/ModulesAndPackages.md)
- [ ] Chapter 20: [Type Hints & Static Typing](python/TypeHintsAndStaticTyping.md)
- [ ] Chapter 21: [Concurrency & Parallelism](python/ConcurrencyAndParallelism.md)
- [ ] Chapter 22: [Context Managers](python/ContextManagers.md)
- [ ] Chapter 23: [Regular Expressions](python/RegularExpressions.md)
- [ ] Chapter 24: [Functional Programming](python/FunctionalProgramming.md)
- [ ] Chapter 25: [Memory Management & Internals](python/MemoryManagementAndInternals.md)
- [ ] Chapter 26: [Metaclasses & Descriptors](python/MetaclassesAndDescriptors.md)
- [ ] Chapter 27: [Testing](python/Testing.md)
- [ ] Chapter 28: [Standard Library Highlights](python/StandardLibraryHighlights.md)
- [ ] Chapter 29: [Networking & Web](python/NetworkingAndWeb.md)
- [ ] Chapter 30: [Database Access](python/DatabaseAccess.md)
- [ ] Chapter 31: [Design Patterns in Python](python/DesignPatterns.md)
- [ ] Chapter 32: [Performance & Profiling](python/PerformanceAndProfiling.md)
- [ ] Chapter 33: [Packaging & Distribution](python/PackagingAndDistribution.md)
- [ ] Chapter 34: [Common Interview Gotchas](python/InterviewGotchas.md)
- [ ] Chapter 35: [Python Versions & Key Features](python/PythonVersions.md)
