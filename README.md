# C-Python-like-Decorators
How to write decorator functions in modern C++14 or higher

Works across MSVC, GNU CC, and Clang compilers

## Skip the tutorial and view the final results 

[https://godbolt.org/z/jSmORB](Tutorial demo) *

[https://godbolt.org/z/zTHveB](practical demo)

*tutorial demo can be found in repo [example.cpp](https://github.com/TheMaverickProgrammer/C-Python-like-Decorators/blob/master/example.cpp)

# The goal
Python has a nice feature that allows function definitions to be wrapped by other existing functions. "Wrapping" consists of taking a function in as an argument and returning a new aggregated function composed of the input function and the wrapper function. The wrapper functions themselves are called decorator functions. 

The language syntax for this to be done automatically for us are simply called "decorators" and look like this:

```python
def stars(func):
  def inner(*args, **kwargs):
      print("**********")
      func(*args, **kwargs)
      print("**********")
      
  return inner
  
# decorator syntax
@stars
def hello_world():
   print("hello world!")
    
# The following prints:
# 
# **********
# hello world!
# **********
hello_world()
```

The decorating function `stars(...)` will take in any kind of input and pass it along to its inner `func` object, whatever that may be. In this case, `hello_world()` does not take in any arguments so `func()` will simply be called.

The `@stars` syntax is sugar for `hello_world = stars(hello_world)`. 

This is a really nice feature for python that, as a C++ enthusiast, I would like to use in my own projects. In this tutorial I'm going to make a close equivalent without using magic macros or meta-object compilation tools.

# Accepting any arbitrary functor in modern C++
Python decorator functions take in a _function_ as its argument. I toyed with a variety of concepts and discovered quickly that lambdas are not as versatile as I had hoped they would be. Consider the following code:

[https://godbolt.org/z/vxEJI3](goto godbolt)

```cpp
template<typename R, typename... Args>
auto stars(R(*in)(Args...)) {
    return [in](Args... args) {
        std::cout << "*******" << std::endl;
        in();
        std::cout << "*******" << std::endl;
    };
}

void hello() {
	cout << "hello, world!" << endl;
}

template<typename R, typename... Args>
auto smart_divide(R(*in)(Args...)) {
    return [in](float a, float b) {
        std::cout << "I am going to divide a and b" << std::endl;

        if(b == 0) {
            std::cout << "Whoops! cannot divide" << std::endl;
            return 0.0f;
        }

        return in(a, b);
    };
}

float divide(float a, float b) {
    return a/b;
}
```

This tries to achieve the following python code:

```python
def smart_divide(func):
   def inner(a,b):
      print("I am going to divide",a,"and",b)
      if b == 0:
         print("Whoops! cannot divide")
         return

      return func(a,b)
   return inner

@smart_divide
def divide(a,b):
    return a/b
```

It works! Great! But try uncommenting line 62. It does not compile. If you looks closely at the compiler output it has trouble deducing the function pointer types from the lambda object returned by the inner-most decorator. This is because **lambdas with capture cannot be converted to function pointers**

If we were to introduce a struct to hold our arbitrary functors, we'd fall into the same problem. By limiting ourselves to a specific expected function-pointer syntax, we lose the ability to accept just about any type we want. The solution is to use a single `template<typename F>` before our decorators to let the compiler know the decorator can take in just about anything we throw at it.

Now we can nest multiple decorator functions together! ... not so fast... 

Now we've lost the type information from `Args...` in our function signature. Luckily there is something we can do about this in C++14 and onward...

# Returning an inner function that can accept any number of arguments
We need to build a function, in our function, that can also accept an arbitrary set of inputs and pass those along to our captured input function. To reiterate, our _returned function_ needs to be able to _forward all the arguments_ to the function we are trying to decorate.

Python gets around this problem by using special arguments `*args` and `**kwargs`. I won't go into detail what these two differerent notations mean, but for our problem task they are equivalent to C++ variadic template arguments. They can be written like so:

```cpp
template<typename... Args>
void foo(Args... args) {
    bar(args...);
}
```

Anything passed into the function are forwarded as arguments for the inner function `bar`. If the types match, the compiler will accept the input. This is what we want, but remember we're returning a function inside another function. Prior to C++14 this might have been impossible to achieve nicely. Thankfully C++14 introduced **template lambdas**

```cpp
return [func]<typename... Args>(Args... args) {
        std::cout << "*******" << std::endl;
        func(args...); // forward all arguments
        std::cout << "\n*******" << std::endl;
    };
```

# Clang and MSVC - `auto` pack for the win
Try toggling the compiler options in godbolt from GNU C Compiler 9.1 to latest Clang or MSVC. No matter what standard you specify, it won't compile. We were so close! Let's inspect further. Some google searches and forum scrolling later, it seems the trusty GCC might be ahead of the curve with generic lambdas in C++14. To quote one forum user:

> C++20 will come with templated and conceptualized lambdas. The feature has already been integrated into the standard draft.

I was stuck on this for quite some time until I discovered this neat trick by trial and error:

```cpp
template<typename F>
auto output(F func) {
    return [func](auto... args) {
        std::cout << func(args...);
    };
}
```

By specifying the arguments for the closure as an `auto` pack, we can avoid template type deduction all together - much more readable!

# Nested decorators
Great, we can begin putting it all together! We have:

* Function that returns function (by lambda closures) (CHECK)
* "Decorator function" that can accept any input function using a template typename (CHECK)
* Inner function can also accept arbitrary args passed from outer function (using auto packs) (CHECK)

Now we can check to see if we can further nest the decorators...

[https://godbolt.org/z/EQnBWM](goto godbolt)

```cpp
// line 64 -- four decorator functions!
auto d = stars(output(smart_divide(divide)));
d(12.0f, 3.0f);
```

output is

```cpp
*******
I am going to divide a=12 and b=3
4
*******
```

First the `stars` decorator is called printing `**********` to our topmost row.
Then the `output` function prints the result of the next function to `cout` so we can see it. The result of the next function is covered by the next two nested functions.

The `smart_divide` function checks the input passed in from the top of the chain `12.0f, 3.0f` if we are dividing by zero or not before forwarding args to the next function `divide` which calculates the result. `divide` returns the result and `smart_divide` returns that result to `output`.

Finally `stars` scope is about to end and prints the last `*********` row

# Works out of the box as-is
Check out line 57 using `printf` 

```cpp
auto p = stars(printf);
p("C++ is %s!", "epic");
```

output is

```cpp
*******
C++ is epic!
*******
```

I think I found my new favorite C++ concept for outputting log files, don't you feel the same way? :)

# Practical examples
There's a lot of debate about C++'s exception handling, lack thereof, and controversial best practices. We can solve a lot of headache by providing decorator functions to let throwable functions fail without fear.

Consider this example: 
[https://godbolt.org/z/zTHveB](goto godbolt)

We can let the function silently fail and we can choose to supply another decorator function to pipe the output to a log file. Alternatively we could also check the return value of the exception (using better value types of course this is just an example) to determine whether to shutdown the application or not.

```cpp
auto read_safe = exception_fail_safe(file_read);

if(!read_safe("missing_file.txt", buff, &sz).compare("OK")) {
    // Whoops! We needed this file. Quit immediately!
    app.abort();
    return;
}
```

# After-thoughts
Unlike python, C++ doesn't let us define new functions on the fly, but we could get closer to python syntax if we had some kind of intermediary functor type that we could reassign e.g.

```cpp
decorated_functor d = smart_divide(divide);

// reassignment
d = stars(output(d));
```

Unlike python, assignment of arbitrary types in C++ is almost next to impossible without type erasure.
We solved this by using templates but every new nested function returns a new type that the compiler sees.
And as we discovered at the beginning, lambdas with capture do not dissolve into C++ pointers as we might expect either. 

With all the complexities at hand, this task non-trivial. 

Additionally, there's no nice syntax at hand for C++ to wrap a function around another at definition without the use of magic macros or mocs. We're left to wrap functions at run-time.

This challenge took about 2 days plugged in and was a lot of fun. I learned a lot on the way and discovered something pretty useful. Thanks for reading!
