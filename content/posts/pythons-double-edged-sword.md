+++
title = 'Is Duck Typing Worth the Cruft?'
summary = "Useful? Yes. Enabling bad practice? Probably."
date = 2024-04-09T18:18:10-04:00
+++

Python's duck typing - or the idea that if an object "talks like a duck and
walks like a duck, it's a duck" is an occasionally helpful functionality that
enables a developer to mock classes with only a subset of their core functionality,
making testing complex classes much easier.

## The Problem

Take for example this class:

```python
class MyClass:

  def __init__():
    ...
    self.load_data()
    self.filter_data()
    self.report_summary()
    self.kickoff_some_process()
    ...

```

With so many methods run on construct, testing this class becomes a bit of a nightmare.
Especially in data intensive applications, we may need to either:

1) Mock up a TON of dummy pandas DataFrames, which is time intensive and logic intensive 
to find comprehensive examples.
2) Start including live data in our tests - which for fast moving, experimental projects
means relying on partially cleaned or changing datasets under your feet, leading to very 
flaky tests.

Even worse - if we have an external function that takes a `MyClass` instance as an argument,
we may have to include all of this dummy data for a test that doesn't even care about it!

```python
def partial_myclass(inst: MyClass):
  ...
  # Maybe we use one attribute of MyClass, but not the others
  # Maybe we call one method, but none of the rest
  return inst.important_attribute + 2
```

## Duck Typing to the Rescue

In theory - this is where duck typing can become useful. Say we define a new `MockMyClass`,
we can create it in such a way that it only implements the attributes or methods that 
are needed by `partial_myclass()`

```python
class MockMyClass:

  def __init__(self, important_attribute: str):
    self.important_attribute = important_attribute

```

And *voila* ~ just like that - we can feed test data into `MockMyClass`, and `partial_myclass`
will accept it with only minor whining from `mypy`.

**The Important Piece is This:** Python doesn't care if the object is truly of a certain type,
as long as it implements the expected attributes/methods

## Where Does This Go Wrong?

We implement our mocked class and life goes on, development continues, tests pass.
Then one day - we get a P1 alert saying our `MyClass` isn't behaving as expected. 
We are *flabbergasted* - because we spent so much time on our beautiful test case!

However, duck typing *may* have lured us into a false sense of security, as our
`MockMyClass` has no ties to the original `MyClass` object anymore. Turns out a junior
developer implemented a change that renames `important_attribute` to `very_important_attribute`!

The whole time - our tests will continue to pass as we did not carry the change into `MockMyClass`.
**In order for duck typing to be effective in testing - the Mocks *must* look like the classes they mock.**


## So What Can We Do?

The easy solution is not robust: "just update the mock when you update the class it mocks!"
This is the dream of small code bases and long development cycles. When push comes to shove, 
this simply will not happen.

Where I think I am right now is *instead* pass the smallest atomic pieces we can into our
functions and methods, consider this change to `partial_myclass` 

```python

# Original
def partial_myclass(inst: MyClass):
  return inst.important_attribute + 2

# New
def partial_myclass(important_attribute):
  return important_attribute + 2
```

Now - `partial_myclass` does not need a duck type at all to test, we merely need a valid `important_attribute`
to pass in and ensure we get the right value returned! However, in larger functions, this could lead to passing
dozens of arguments to functions, which is decidedly [not Clean, according to Uncle Bob.](https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882)
Thus - I believe if we truly have 10 arguments to our function, we are testing the wrong part of the code. 
Assuming we do not use all 10 arguments at once, we can pass the first three to a subfunction, test those, and  
go from there.

## Takeaways:

* Duck Typing is powerful, but can create disconnects between your mocks and the objects they are there to mock
* When possible - do not pass unused attributes to functions.

## Follow ups I need to do:

I feel like this case must be addressed in Clean Code - but I have not read it (despite owning a copy). 
I am going to read at least the first few chapters.
