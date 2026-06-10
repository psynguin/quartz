> [!abstract] What We'll Cover
> Welcome to Chapter 2! If you've been writing Python for a while, you're already intimately familiar with objects—you just might not have built your own blueprints yet. In this chapter, we're going to transition from functional thinking (passing raw data into functions) to Object-Oriented thinking (bundling state and behavior together). We'll cover how to define your very first classes, how to breathe life into them with `__init__`, and why the `self` keyword is your explicit guide to instance state.

## Defining Classes and Instantiating Objects

In Python, everything is an object. When you create a `list`, a `dict`, or even a simple `str`, you are interacting with objects that Python has kindly built for you. Creating a class is simply defining a new, custom data type.

A **class** acts as a blueprint or factory. It establishes the structure, attributes, and behaviors that its objects will have. An **object** (often called an **instance**) is the concrete manifestation of that blueprint taking up space in your computer's memory.

> [!info] Naming Conventions
> Following PEP 8, Python's official style guide, class names should always use `CamelCase` (also known as `CapWords`). This makes them instantly visually distinct from `snake_case` functions and variables.

Let's look at how to define and create your first class. Notice how instantiating an object looks exactly like calling a function.

> [!example] Defining and Instantiating a Class
```python
# Defining a minimal class blueprint
class RequestHandler:
    pass  # 'pass' is used as a placeholder for an empty block

# Instantiating objects (creating instances from the blueprint)
handler1 = RequestHandler()
handler2 = RequestHandler()

print(type(handler1)) 
# Output: <class '__main__.RequestHandler'>

print(handler1 is handler2) 
# Output: False (They are distinct, independent objects in memory!)
```

Unlike some other languages, Python doesn't require massive amounts of boilerplate. A class can be as simple as you need it to be, acting as a handy namespace to group related concepts.

## The `__init__` Constructor Method

Right now, our `RequestHandler` doesn't *do* much. To make objects useful, we need to initialize them with some starting state. Enter the `__init__` method.

> [!tip] Constructor vs. Initializer
> While developers colloquially call `__init__` the "constructor", it is technically an **initializer**. When you call `MyClass()`, Python first secretly calls `__new__` to allocate memory and construct the object, and *then* hands that fresh object over to `__init__` to initialize its attributes.

The `__init__` method runs automatically the moment you instantiate the class. The double underscores (`__`) are a Python convention indicating a "dunder" (double underscore) or "magic" method. These are reserved hooks that Python calls behind the scenes.

> [!example] Using `__init__`
```python
class DatabaseConnection:
    # __init__ allows us to pass arguments during instantiation
    def __init__(self, host, port):
        print(f"Initializing connection to {host}:{port}")

# The arguments inside the parentheses are passed directly to __init__
db = DatabaseConnection("localhost", 5432)
```

But wait, what is that `self` parameter doing there? Let's break down one of Python's most defining characteristics.

## The Crucial Role of `self`

If you're coming from languages like Java, C++, or JavaScript, you might be used to a magical `this` keyword that implicitly points to the current object. Python firmly rejects that magic. 

> [!note] The Zen of Python: Explicit is Better Than Implicit
> Python requires `self` to be explicitly declared as the *first parameter* of all instance methods. `self` is simply a reference to the specific instance of the class that is currently being operated on.

When you call a method on an object, like `db.connect()`, Python internally translates this to `DatabaseConnection.connect(db)`. The object itself (`db`) is automatically passed as the first positional argument, which neatly maps to the `self` parameter in your method definition.

> [!warning] `self` is a Convention, Not a Keyword!
> `self` is not a reserved keyword in Python; it's a strictly adhered-to naming convention. You *could* technically name it `this`, `me`, or `potato`, but doing so would violate PEP 8 and earn you the ire of every other Python developer who reads your code. Stick to `self`.

Because `self` is explicit, scoping rules are remarkably clear. You never have to guess whether a variable inside a method is a temporary local variable or an instance attribute—instance attributes are *always* prefixed with `self.`. 

## Instance Attributes and Instance Methods

Now that we understand `self`, we can bring state and behavior together. 

*   **Instance Attributes:** These are variables bound to a specific object, representing its *state*. You define them by assigning a value to `self.attribute_name`, typically inside the `__init__` method.
*   **Instance Methods:** These are functions defined within the class that operate on the object's instance attributes, representing its *behavior*. They must always take `self` as their first parameter.

Let's tie it all together by building a `TextAnalyzer` class. Notice how we use `self` to clearly distinguish between local method variables and the persistent state of the object.

> [!example] State and Behavior in Action
```python
class TextAnalyzer:
    def __init__(self, text):
        # Instance attributes define the state of THIS specific object
        self.text = text
        self.word_count = 0 
        self.is_analyzed = False

    # Instance method: modifies or uses the instance's state
    def analyze(self):
        # 'words' is a local variable; it doesn't need 'self'
        words = self.text.split()
        
        # We MUST use 'self' to update the persistent instance state
        self.word_count = len(words)
        self.is_analyzed = True
        return self.word_count

    def get_summary(self):
        # We can also call other methods using 'self' if needed
        if not self.is_analyzed:
            return "Text has not been analyzed yet."
        return f"Analyzed {self.word_count} words."

# 1. Instantiation
analyzer = TextAnalyzer("OOP in Python is explicit and powerful.")

# 2. Accessing instance attributes directly
print(analyzer.is_analyzed) 
# Output: False

# 3. Calling instance methods
analyzer.analyze()
print(analyzer.get_summary()) 
# Output: Analyzed 7 words.
```

> [!info] A Note on Privacy
> In the code above, we accessed `analyzer.is_analyzed` directly from outside the class. Unlike languages that strictly enforce `private` and `public` variables, Python operates on a "we are all consenting adults here" philosophy. Everything is public by default. We'll learn how to signal privacy and protect attributes in later chapters.

By using classes to bundle state (`self.text`, `self.word_count`) with the behaviors that modify that state (`analyze()`, `get_summary()`), we create clean, modular, and understandable code. You're no longer just passing raw variables into disjointed functions—you are orchestrating interactions between fully-fledged objects!
