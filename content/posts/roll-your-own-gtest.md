---
title: "Roll Your Own GTest"
date: 2025-02-07T22:22:37+01:00
author: Martin Kauppinen
---

Something I keep telling myself and others about when it comes to programming is
"_There is no such thing as magic_". By this I mean that anything you encounter
in the world of programming _can_ be understood. And you should try to make an
effort to understand things that you currently don't.

One such thing I've been thinking about is the
[`GoogleTest`](https://github.com/google/googletest) framework for writing tests
in C++. On the face of it, it's pretty simple. You define a test suite as a
class with all the data and setup needed for your suite of tests, and with some
macros you instantiate a bunch of tests:

```cpp
#include <gtest/gtest.h>

class FooSuite : public ::testing::Test
{
    // test suite data goes here
};

TEST_F(FooSuite, Foo)
{
    // test some stuff
}

TEST_F(FooSuite, Bar)
{
    // test some more stuff
}
```

And in your `main` function you just do something like

```cpp
#include <gtest/gtest.h>

int main(int argc, char **argv)
{
    ::testing::InitGoogleTest(&argc, argv);
    return RUN_ALL_TESTS();
}
```
You don't even have to write this main test runner yourself, `GoogleTest`
provides a program like this that you can simply add as a linked library in your
`CMakeLists.txt`. This will run every `TEST_F` macro you defined earlier and if
they all pass, you're happy. Really simple stuff.

But there's something hidden here. How does `GoogleTest` actually find all the
tests you defined? You never explicitly call them anywhere. What would you even
call? What _magic_[^magic] is happening here that they are called automatically
simply by defining them? Does the test executable somehow open itself, getting
addresses of every constructor and test body through
[`dlsym(3)`](https://man7.org/linux/man-pages/man3/dlsym.3.html) or something?
[^magic]: No such thing!

As it turns out, there is no such thing as magic. The trick is actually pretty
simple.

Some code blocks below will have a path attached to them. They are relative
paths into the [companion repo](https://github.com/martinkauppinen/smalltest) to
this post.

## The knowledge of the secrets of Nature
To illustrate what is going on behind the scenes, here's a little example
program featuring everyone's favorite class: `Widget`!

{{< labelled-highlight lang="cpp" filename="examples/01-widget.cpp" >}}
#include <iostream>

class Widget
{
public:
    Widget() { std::cout << "Widget constructor" << std::endl; }
    ~Widget() { std::cout << "Widget destructor" << std::endl; }
};

int main()
{
    std::cout << "Hello, world!" << std::endl;
    auto widget = Widget{};
    return 0;
}
{{< /labelled-highlight >}}

If we compile and run this little program, we get the following expected output:
```bash
$ make 01-widget && ./01-widget
Hello, world!
Widget constructor
Widget destructor
```

No surprises here. In `main`, we print "Hello, world!", then the `Widget`
constructor is called when we create our widget, which prints "Widget
constructor". Then as the program exits and cleans up its stack, the `Widget`
destructor is called, printing "Widget destructor". Simple stuff.

Now watch what happens if I move the widget outside the `main` function, into
the global scope.

{{< labelled-highlight lang="cpp" filename="examples/02-widget.cpp"
options="hl_lines=10-10">}}
#include <iostream>

class Widget
{
public:
    Widget() { std::cout << "Widget constructor" << std::endl; }
    ~Widget() { std::cout << "Widget destructor" << std::endl; }
};

auto widget = Widget{};

int main()
{
    std::cout << "Hello, world!" << std::endl;
    return 0;
}
{{< /labelled-highlight >}}

Compile and run:

```bash
$ make 02-widget && ./02-widget
Widget constructor
Hello, world!
Widget destructor
```

"Widget constructor" prints _before_ "Hello, world!"! This means that we ran
code _before_ entering `main`. This is because variables in non-local variables
with static storage duration (such as our global `widget`) are initialized as
part of program startup. _Before entering `main`_[^cpp-startup].
[^cpp-startup]: https://en.cppreference.com/w/cpp/language/initialization#Non-local_variables

This is the trick that makes `GoogleTest` tick, and why you don't have to
manually call the tests you define. Their constructors register themselves under
the hood as instances of non-local variables to whatever handles the actual
execution of tests, which itself can be a non-local variable such that
everything is set up automatically.

The second example can actually be simplified even further by having our static
global be defined as part of the class definition:

{{< labelled-highlight lang="cpp" filename="examples/03-widget.cpp"
options="hl_lines=8-8">}}
#include <iostream>

class Widget
{
public:
    Widget() { std::cout << "Widget constructor" << std::endl; }
    ~Widget() { std::cout << "Widget destructor" << std::endl; }
} widget;

int main()
{
    std::cout << "Hello, world!" << std::endl;
    return 0;
}
{{< /labelled-highlight >}}

This compiles and prints exactly the same as example 02.

## Rolling your own

Now `GoogleTest` has _a lot_ of features and to keep this post to a reasonable
length, I'm going to implement a much simpler unit testing framework inspired by
`GoogleTest`. But it will only be able to define test cases. No test suites,
parameterized test suites, type-parameterized test cases, etc. Extremely simple
stuff. And only a couple of helper macros to use for tests. Specifically:

1. `UNIT_TEST(test_name)`: The main macro defining a test
2. `EXPECT(expr)`: Expects `expr` to be true. Test fails but continues if it's
   false.
3. `ASSERT(expr)`: Expects `expr` to be true. Test fails and terminates if it's
   false.

I will also not take special care to do everything "_properly_". This is a
learning thing, not production code.

### The basics
Let's start with a base class called `UnitTest`. I'll put this in a namespace
called `smalltest` as the name of this micro-framework:


{{< labelled-highlight lang="cpp" filename="src/smalltest.hpp" >}}
#pragma once

#include <string>

namespace smalltest {

class UnitTest
{
protected:
    virtual void Run() = 0;
};

}
{{< /labelled-highlight >}}

It's an abstract class with a single method, `Run()`, which is what the user
will define as the main body of the test. Every instance of this class (every
call to `UNIT_TEST(test_name)`) will define a class that inherits from this base
class, creates a static global, which calls the constructor that registers the
test in the test executor.

### The registry
Now we need a central registry to hold all the tests and execute them when it's
time. And there should only be one instance of this class. And it should be
static-initialized. Sounds like a singleton to me. Let's call it `TestRegistry`

{{< labelled-highlight lang="cpp" filename="src/smalltest.hpp" >}}
namespace smalltest {
// ---<snip>---
class TestRegistry
{
public:
    static TestRegistry *GetInstance()
    {
        if (instance_ == nullptr)
        {
            instance_ = new TestRegistry{};
        }
        return instance_;
    }
private:
    TestRegistry() {}
    static TestRegistry *instance_;
};
}
{{< /labelled-highlight >}}

Simple singleton. Now we need to hold on to all `UnitTest` objects somehow. How
about each test having a name that we map from a pointer to the test instance,
and also keeping track of whether the test passed?

{{< labelled-highlight lang="cpp" filename="src/smalltest.hpp"
options="hl_lines=13-14 5-8">}}
class TestRegistry
{
public:
    // ---<snip>---
    void RegisterTest(UnitTest *test, std::string name)
    {
        tests_.insert({test, {name, true}});
    }

private:
    TestRegistry() {}

    using TestData = std::tuple<std::string, bool>;
    std::map<UnitTest*, TestData> tests_{};

    static TestRegistry *instance_;
};
}
{{< /labelled-highlight >}}

No magic here. But let's get back to the trick now.

### The main macro
We want the `UNIT_TEST(test_name)` macro to expand to a class derived from
`UnitTest` that instantiates a static global and registers itself in to the
`TestRegistry` in the constructor. Basically we want it to expand to something
like this:


```cpp
class test_name : public ::smalltest::UnitTest
{
public:
    test_name() : name_{"test_name"}
    {
        auto* registry =
            ::smalltest::TestRegistry::GetInstance();
        registry->RegisterTest(name_, this);
    }
protected:
    void Run() override; // to be defined
private:
    std::string name_;
} test_name; // here is the instantiation trick
```

And since we want to use it like
```cpp
UNIT_TEST(foo)
{
    EXPECT(true);
}
```
the last line of the macro should implement the `Run` method. Some minor tweaks
to the above gets the macro implemented quickly:


{{< labelled-highlight lang="cpp" filename="src/smalltest.hpp" >}}
#define UNIT_TEST(test_name)                           \
  class test_name : public ::smalltest::UnitTest       \
  {                                                    \
  public:                                              \
      test_name() : name_{ #test_name }                \
      {                                                \
          auto* registry =                             \
              ::smalltest::TestRegistry::GetInstance();\
          registry->RegisterTest(name_, this);         \
      }                                                \
  protected:                                           \
      void Run() override;                             \
  private:                                             \
      std::string name_;                               \
  } test_name;                                         \
  void test_name::Run() // the "magic" line (no such thing!)
{{< /labelled-highlight >}}

Now the macro instantiates a new `UnitTest`-derived class that automatically
registers itself in the `TestRegistry`, and the final line means that the block
following the macro is the definition of the `Run` method.

Now we need to make the `TestRegistry` execute all the `Run` methods. To do
this, it first needs to become a friend of `UnitTest`, since I figured `Run`
should be a protected method:

{{< labelled-highlight lang="cpp" filename="src/smalltest.hpp"
options="hl_lines=5">}}
class UnitTest
{
protected:
    virtual void Run() = 0;
    friend class TestRegistry;
};
{{< /labelled-highlight >}}

And running the test is a simple for-each loop:

{{< labelled-highlight lang="cpp" filename="src/smalltest.hpp"
options="hl_lines=5-24">}}
class TestRegistry
{
public:
    // ---<snip>---
    void RunTests()
    {
        size_t failed_tests = 0;
        for (auto& [test, test_data] : tests_)
        {
            auto& [name, passed] = test_data;
            test->Run();

            if (!passed)
            {
                failed_tests++;
            }
        }

        if (failed_tests == 0)
        {
            return EXIT_SUCCESS;
        }
        return EXIT_FAILURE;
    }
private:
    // ---<snip>---
};
}
{{< /labelled-highlight >}}

Of course, a test case needs to be able to inform the registry that it has
failed. So let's add a little method for that as well:

{{< labelled-highlight lang="cpp" filename="src/smalltest.hpp"
options="hl_lines=5-9">}}
class TestRegistry
{
public:
    // ---<snip>---
    void FailTest(UnitTest *test)
    {
        auto& [name, passed] = tests_[test];
        passed = false;
    }
private:
    // ---<snip>---
};
}
{{< /labelled-highlight >}}

And we're almost done, actually! We just need a couple of small details.

### The helper macros
I mentioned that we wanted `EXPECT(expr)` and `ASSERT(expr)` calls inside the
tests to cause them to fail (and optionally abort the test). Since they will be
called inside the implementation of the `Run` method, the aborting behavior can
be implemented with a simple `return` statement. Otherwise the macros are
completely identical.

{{< labelled-highlight lang="cpp" filename="src/smalltest.hpp" >}}
#define EXPECT(expr)                                     \
    do                                                   \
    {                                                    \
        if (!(expr))                                     \
        {                                                \
            auto registry =                              \
                ::smalltest::TestRegistry::GetInstance(); \
            registry->FailTest(this);                    \
        }                                                \
    } while (false)

#define ASSERT(expr)                                     \
    do                                                   \
    {                                                    \
        if (!(expr))                                     \
        {                                                \
            auto registry =                              \
                ::smalltest::TestRegistry::GetInstance(); \
            registry->FailTest(this);                    \
            return;                                      \
        }                                                \
    } while (false)
{{< /labelled-highlight >}}

Here I'm using the [do-while(false)](https://stackoverflow.com/a/923833) trick
to make them behave more like function statements. Now the functionality is
complete! The only thing left is initialization.

### The init
The whole machinery kicks off when we define our test runner program as follows:
```cpp
#include "smalltest.hpp"

smalltest::TestRegistry *smalltest::TestRegistry::instance_
    = nullptr;

int main()
{
    auto *registry = smalltest::TestRegistry::GetInstance();
    return registry->RunTests();
}
```

That `nullptr` line is needed because non-const static members must be
initialized out of line.

In fact, since this is what every test running executable using smalltest will
look like, we can provide a convenience macro for it:

{{< labelled-highlight lang="cpp" filename="src/smalltest.hpp" >}}
#define RUN_SMALLTEST()                                     \
smalltest::TestRegistry *smalltest::TestRegistry::instance_  \
    = nullptr;                                             \
int main()                                                 \
{                                                          \
    auto *registry = smalltest::TestRegistry::GetInstance();\
    return registry->RunTests();                           \
}
{{< /labelled-highlight >}}

And it can be used! Here's a simple little program:

{{< labelled-highlight lang="cpp" filename="examples/footest.cpp" >}}
#include "smalltest.hpp"
#include <iostream>

UNIT_TEST(foo)
{
    EXPECT(true);
    std::cout << "This is fine" << std::endl;
}

UNIT_TEST(bar)
{
    EXPECT(false);
    std::cout << "This is less fine" << std::endl;
}

UNIT_TEST(baz)
{
    ASSERT(false);
    std::cout << "This will never print" << std::endl;
}

RUN_SMALLTEST();
{{< /labelled-highlight >}}

If we compile and run this program:
```bash
$ make footest && ./footest
This is fine
This is less fine

$ echo $?
1
```
Yeah, not much happens. But the exit code was non-zero! So we have failing
tests, which is to be expected, since both `bar` and `baz` are expecting
`false` to be `true`, which I don't have to tell you will never be the case.

### The finishing touch
We did save the name of each test. Let's simply print out which test is about to
run, and at the end of all tests print how many passed and which ones failed.

{{< labelled-highlight lang="cpp" filename="src/smalltest.hpp"
options="linenos=inline,hl_lines=7-7 9-9 13-16 22-25 30-34 38-42 49-53">}}
class TestRegistry
{
public:
    // ---<snip>---
    void RunTests()
    {
        size_t passed_tests = 0;
        size_t failed_tests = 0;
        std::vector<std::string> fails{};
        for (auto& [test, test_data] : tests_)
        {
            auto& [name, passed] = test_data;
            std::cout
                << "RUN: "
                << name
                << std::endl;
            test->Run();

            if (passed)
            {
                passed_tests++;
                std::cout
                    << "PASS: "
                    << name
                    << std::endl;
            }
            else
            {
                failed_tests++;
                fails.emplace_back(name);
                std::cout
                    << "FAIL: "
                    << name
                    << std::endl;
            }
        }

        std::cout
            << "\nTests run:   " << tests_.size() 
            << "\nTests passed:" << passed_tests
            << "\nTests failed:" << failed_tests
            << std::endl;

        if (failed_tests == 0)
        {
            return EXIT_SUCCESS;
        }

        std::cout << "\nFailed tests:\n";
        for (const auto& test : fails)
        {
            std::cout << "\t" << test << "\n";
        }
        return EXIT_FAILURE;
    }
private:
    // ---<snip>---
};
}
{{< /labelled-highlight >}}

Now if we compile and run `footest.cpp` from before:

```bash
$ make footest && ./footest
RUN: foo
This is fine
PASS: foo
RUN: bar
This is less fine
FAIL: bar
RUN: baz
FAIL: baz

Tests run:    3
Tests passed: 1
Tests failed: 2

Failed tests:
        bar
        baz
```

And look at that! We have a barebones unit testing framework!

## Wrapping up
All code blocks from above have paths relative to [this companion
repo](https://github.com/martinkauppinen/smalltest). It contains all the
examples from this post and a couple extras. The entire framework is just above
100 lines of C++, contained in a single header file (`src/smalltest.hpp`), which
makes it really simple to use for any testing purpose. This is not the only
small C++ testing framework, far from it. There are multiple projects that
implement this exact thing. And of course, `GoogleTest` is the most complex
example. But this barebones demonstration contains just enough functionality to
be understandable and extendable to implement features from any other similar
testing framework you might find.

Hopefully you learned something, dear reader. I know I did. And here's a short,
incomplete list of features to implement if the mood hits you to do something
similar based on this:

* More `EXPECT_` and `ASSERT_` helper macros
* Prettier printing (e.g. in the `EXPECT` and `ASSERT` macros)
* Exception handling
* Proper test suites, like `GoogleTest`.
* Test shuffling
* Threading

The main takeaway?

> There is no such thing as magic, though there is such a thing as a knowledge
> of the secrets of Nature.
>
> - H. Rider Haggard, [from the book "__She__"](https://www.gutenberg.org/files/3155/3155-h/3155-h.htm)

(I have to admit, I have not actually read the book. I just really like the
quote, as it summarizes my feelings towards topics in programming that seem
mysterious at first.)

