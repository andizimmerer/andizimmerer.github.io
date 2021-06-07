---
title: "Header-Only Self-Registering Classes for Factories in C++"
date: 2020-05-02
summary: "Sometimes, especially with the factory pattern, you want your derived classes to register automatically to a factory class.
However, I found this harder than expected! Especially if you want it to be header only. Here is a quick walk-through of how this can be done."
---


# Background: Serialization of Derived Classes

Recently, I was working on a [benchmarking framework for block partitioning in databases (currently private, but will be released anytime soon)][blockpartitioning] and I came across the problem of (de-)serializing user-defined derived classes of a base class in C++. While serializing is not too much of a problem (e.g. define an abstract function `virtual void serialize(std::ostream& out) = 0` in the base class and implement the function in the derived classes), deserialization is much harder.
Ultimately, I wanted every derived class to implement its own (de-)serialization functions for potentially complex types. But choosing the _correct_ deserialization routine when reading a serialized file is tricky.

To borrow the example from [the excellent ISO CPP article][ISOCPP], here is some code that visualizes our problem:

We have an abstract base class `Shape` and a derived class `Rectangle`.
Our goal is to correctly deserialize all derived classes of `Shape` by picking the _correct_ concrete implementation. If we pick the _wrong_ deserialization function (e.g. `Circle` instead of `Rectangle`), we will end up with garbage.

```cpp
class Shape {
 public:
  static void serialize(std::ostream& out, Shape& shape) {
    shape.serialize(out);  // Forward to the derived implementation.
  }

  static std::unique_ptr<Shape> deserialize(std::istream& in) {
    // How should we know how to correctly deserialize the object?
    // In other words: which object do we even want to create from the stream??
  }

 protected:
  virtual void serialize(std::ostream& out) = 0;
};

class Rectangle : public Shape {
 public:
  void serialize(std::ostream& out) override {
    // Write properties of the current Rectangle to the output stream.
  }

  static std::unique_ptr<Shape> deserialize(std::istream& in) {
    // Create a new Rectangle from the input stream.
  }
 private:
  int64_t width_, height_;
};
```

**Some notes on the example:**

Generally speaking, this strongly resembles the ["Factory Method Pattern"][factorypattern], but combines the abstract _Factory/Creator_ and the _Interface/Product_ into one class (`Shape`) and also combines the concrete implementations of _RectangleFactory_ and the concrete `Shape` _Rectangle_ into one class. It is left to the reader to figure out which function belongs to which actor in the factory pattern.

Further, I personally prefer two static methods `serialize()` and `deserialize()` in the `Shape` class so that both functions have a similar interface (although `serialize()` could also be non-static).

In the following sections we will gradually build a nice solution for this and discuss why our approach works.
And for the busy reader, there will be a complete code example in the end 😘

# Thoughts on a Solution

The general idea of the solution behind this problem is easily explained:
When serializing, write some sort of _"class identifier"_ to the file before the actual data of the object.
This might be the name of the derived class or some sort of (user supplied) class identifier.
When deserializing again, we first read the class identifier and pick the correct derived class based on that identifier.
Then we call the deserialization function for this particular class, which will now handle the correct deserialization of the object.

A very good starting point is the [extensive description from ISO CPP about serialization][ISOCPP].
It handles many caveats over serialization and also covers our current problem!
However, our goal is to do this as nicely as possible.

{{< notice note >}}
If you are by any chance interested in how the same problem can be nicely solved in Rust,
have a look at [typetag](https://github.com/dtolnay/typetag) from the developer of [serde](https://github.com/serde-rs/serde).
This is a great solution and is very similar to our approach.
{{< /notice >}}

In the previous paragraph I mentioned that we "pick the correct serialization function".

But **how can a correct mapping between "class identifier" and the corresponding class be achieved?**

Without loss of generality, we will pick the name of the derived class as a class identifier.
Usually, there will be a `static std::map<K, V>` from class name to class prototypes of the derived class or function pointers that will take the input stream and produce an object of the correct derived class.
I like the idea of functions more because then we don't have "partly constructed objects only used to create a full object" laying around.
The signature of the deserialization function's pointer would look like `std::unique_ptr<Shape> (*)(std::istream&)`.

For this, we can put a `static std::map<std::string, Shape* (*)(std::istream&)>` into the `Shape` class.
To handle things nicely, we can alter the functions in the `Shape` class:

```cpp
class Shape {
  // <snip>

  static void serialize(std::ostream& out, Shape& shape) {
    // Before we serialize the actual object,
    // write the class identifier to the output stream.
    const std::string& class_id = shape.class_identifier();
    out.write(class_id.c_str(), class_id.size() + 1 /*terminating null*/);

    shape.serialize(out);  // Forward to the derived implementation.
  }

  static std::unique_ptr<Shape> deserialize(std::istream& in) {
    // Read the class identifier.
    std::string class_id;
    std::getline(in, class_id, '\0');

    // Pick the correct deserialization function from the type registry
    // by looking at the class identifier.
    auto deserialize_func = type_registry[class_id];
    std::unique_ptr<Shape> derived_object = deserialize_func(in);
    return derived_object;
  }

  // Type registry which maps class identifiers to deserialization functions,
  // which produce an object of the corresponding derived class.
  static std::map<std::string, std::unique_ptr<Shape> (*)(std::istream&)> type_registry_;
};
```

Nice, this looks already quite good!
Neither the serialization function nor the deserialization function of the derived class needs to know that they are part of this weird problem! 😄

While this is already quite nice, one big open question is: how do we _reliably_ populate the `type_registry_`?
Of course, we can do this per hand and write down every derived class... okay, just kidding 😆  
As soon as someone implements a new derived class and forgets to alter the contents _at a different location_, we can't read the file anymore.

We are dealing with a more general problem here. Remember when I said that the structure closely resembles the factory pattern?
What we want is **self-registering types in a factory**!


# Self-Registering Classes in a Factory

Luckily, there were people out there who [already looked][bfilipek] at [this problem][sacko87] in the past.
While this solution is largely based on the ideas of these two sources, I go one step further:
Solve some problems from the previous approaches and also do it header-only!
And you even get a complete and running example in the end for free! Excited?
Let's get started!

Okay, so the idea is to create a `static bool register_type(...)` function in the `Shape` class that can be called from the derived classes and adds one entry to the `type_registry`.

```cpp
class Shape {
  // <snip>

  using deserialize_func = std::unique_ptr<Shape> (*)(std::istream&);

  static bool register_type(const std::string& class_id,
                            const deserialize_func factory_method) {
    if (type_registry_.find(class_id) != type_registry_.end()) {
      std::cerr << "Class ID " << class_id << " is already used." << std::endl;
      return false;
    }
    type_registry_.insert(std::make_pair(class_id, factory_method));
    return true;
  }

  static std::map<std::string, deserialize_func> type_registry_;
};
```

Nice, looks useful!
But think about it: How does this even help? How should we call this function? In the constructor of the derived class?
This won't really help, because then types are only registered when at least one object has been created before.

The idea is to call the `register_type()` function from a _static context inside the derived class_!
Why? Because then it will be called when _static objects_ are initialized at program startup and this ensures that the function is called before any useful work is done! Nice!
How can we do this? By creating a `static` field inside each derived class that gets assigned the returned value from `register_type()`.

More precisely, this would look like:
```cpp
class Rectangle : public Shape {
  static std::unique_ptr<Shape> deserialize_rectangle(std::istream& in) {
    // ...
  }

  static const bool registered_;
};

// Usually we would do this in the .cpp file, but we wanted it to do header-only, right?
const bool Rectangle::registered_ = Shape::register_type("Rectangle", &deserialize_rectangle); 
```

When the program is started, one of the first steps is to initialize static objects.
Even if `registered_` appears to be unused, we can be sure that it won't be optimized away, because this is [forbidden by the C++ standard][static-elimination].

Thus, `registered_` will be initialized by calling the `register_type()` function and our map in the `Shape` class is being automatically populated at program startup, right?

Sadly, not necessarily. The chances that you end up with an uninitialized map are quite high.
And even worse, this will not just result in  types not being registered, but rather in a program crash.
The problem is that we run into the [_"static initialization order fiasco"_][static-fiasco] because the order in which static values are initialized is undefined when they are in different [translation units][translation-unit].

One of the [original posts][bfilipek] mentioned that this wouldn't happen because of ["Zero initialization"][zero-initialization], but this sadly won't help us if the two classes are in different translations units, which they probably are.

# Circumventing the Static Initialization Order Fiasco

But luckily, there is [a solution to this problem][fiasco-solution]:
If the classes are in different translation units, we can _force_ the initialization of the `type_registry_` by putting it inside a function call and call this function instead.
How? By making use of `static` variables inside functions in our `Shape` class.
This is also called the _Construct On First Use Idiom_.

```cpp
class Shape {
  // <snip>

 private:
  static std::map<std::string, std::unique_ptr<Shape> (*)(std::istream&)>& get_type_registry() {
    // Static: One and the same instance for all function calls.
    static std::map<std::string, std::unique_ptr<Shape> (*)(std::istream&)> type_registry;
    return type_registry;
  }
};
```

Nice! Now we even know how to deal with random static initialization order!
Putting it all together, we end up with the following code:

# Complete Solution

```cpp
#include <iostream>
#include <map>
#include <memory>
#include <string>

class Shape {
  // Function pointer that takes a stream and produces an object of a
  // derived class.
  using deserialize_func = std::unique_ptr<Shape> (*)(std::istream&);

 public:
  virtual ~Shape() = default;

  // Provide a high-level serialization function that stores the
  // class_identifier.
  static void serialize(std::ostream& out, Shape& shape) {
    const std::string& class_id = shape.class_identifier();
    out.write(class_id.c_str(), class_id.size() + 1 /*terminating null*/);
    shape.serialize(out);
  }

  // Provide a high-level deserialization function
  // that dispatches to the correct deserialization function based
  // on class_identifier.
  static std::unique_ptr<Shape> deserialize(std::istream& in) {
    std::string class_id;
    std::getline(in, class_id, '\0');
    // Look up the class_id in the type_registry
    // and call the proper deserialization function.
    return get_type_registry()[class_id](in);
  }

 protected:
  // Return the identifier of the derived class.
  virtual std::string class_identifier() const = 0;
  // Serialize the derived class to a stream.
  virtual void serialize(std::ostream& out) const = 0;

  // This function needs to be called by a derived class inside a 
  // *static* context.
  static bool register_type(const std::string& class_id,
                            const deserialize_func factory_method) {
    if (get_type_registry().find(class_id) != get_type_registry().end()) {
      std::cerr << "Class ID " << class_id << " is already used." << std::endl;
      return false;
    }
    get_type_registry().insert(std::make_pair(class_id, factory_method));
    return true;
  }

 private:
  // Prevent "static initialization order fiasco".
  static std::map<std::string, deserialize_func>& get_type_registry() {
    static std::map<std::string, deserialize_func> type_registry;
    return type_registry;
  }
};

class Rectangle : public Shape {
 public:
  std::string class_identifier() const override { return "Rectangle"; }

  void serialize(std::ostream& out) const override {
    // ...
  }

  static std::unique_ptr<Shape> deserialize_rectangle(std::istream& in) {
    // ...
  }

 private:
  static const bool registered_;

  int64_t width_, height_;
};

// Statically register the "Rectangle" class to the factory.
const bool Rectangle::registered_ = Shape::register_type("Rectangle", &deserialize_rectangle);
```


# Further Work

One obvious downside of this approach is that derived classes need to be registered manually.
It would be much nicer if they could be _forced_ to register.
Following up this blog post, [Alexander van Renen][alexvanrenen] proposed a solution to this problem by introducing a new `ShapeInterface`.
Check out [his solution on GitHub][github-alexvanrenen]!


[blockpartitioning]: https://github.com/andreaskipf/blockpartitioning
[ISOCPP]: https://isocpp.org/wiki/faq/serialization#serialize-inherit-no-ptrs
[factorypattern]: https://en.wikipedia.org/wiki/Factory_method_pattern
[bfilipek]: https://www.bfilipek.com/2018/02/factory-selfregister.html
[sacko87]: https://gist.github.com/sacko87/3359911
[static-fiasco]: http://www.cs.technion.ac.il/users/yechiel/c++-faq/static-init-order.html
[fiasco-solution]: http://www.cs.technion.ac.il/users/yechiel/c++-faq/static-init-order-on-first-use.html
[static-elimination]: http://eel.is/c++draft/basic.stc.static#2
[translation-unit]: https://www.tutorialspoint.com/What-is-a-translation-unit-in-Cplusplus
[zero-initialization]: https://en.cppreference.com/w/cpp/language/zero_initialization
[github-alexvanrenen]: https://gist.github.com/alexandervanrenen/c09af12a075a53eba9b6a07d992ed8db
[alexvanrenen]: https://www.linkedin.com/in/alexander-van-renen-5133a280/