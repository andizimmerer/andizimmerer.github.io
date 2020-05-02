---
title: "Header-Only Self-Registering Classes for Factories in C++"
date: 2020-05-02
summary: "Sometimes, especially with the factory pattern, you want your derived classes to register automatically to a factory class.
However, I found this harder than expected! Especially if you want it to be header only. Here is a quick walk-through of how this can be done."
---


# Background: Serialization of Derived Classes

Recently, I was working on a [benchmarking framework for block partitioning in databases][blockpartitioning] and I came across the problem of (de-)serializing user-defined derived classes of a base class in C++. While serializing is not too much of a problem (e.g. define a abstract function `virtual void serialize(std::ostream& out) = 0` in the base class and derived classes need to implement it), deserialization is much harder.
Ultimately, I wanted every derived class to implement their own (de-)serialization functions for potential complex types. But picking the _correct_ deserialization routine when reading a file is tricky.


To borrow the example from [ISO CPP][ISOCPP], here is some code that visualizes our problem:

```cpp
class Shape {
 public:
  static void serialize(std::ostream& out, Shape* shape) {
    shape->serialize(out);  // Forward to the derived implementation.
  }
  static Shape* deserialize(std::istream& in) {
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

  static Shape* deserialize(std::istream& in) {
    // Create a new Rectangle from the input stream.
  }
 private:
  int64_t width_, height;
};
```

#### Some notes on the example:

I personally prefer the two static methods `serialize()` and `deserialize()` in the `Shape` class so that both functions have a similar interface.
Strictly speaking, this strongly resembles the ["factory pattern"][factorypattern], but combines the abstract _Factory_ and the _Interface_ into one class (`Shape`) and also combines the concrete implementations of _RectangleFactory_ and the concrete `Shape` _Rectangle_ into one class. It is left to the reader to figure out which function belongs to which actor in the factory pattern.

In the following sections we will gradually build a nice solution for this and discuss why this works.
And for the busy reader, there will be a complete code example in the end 😘

# Solution for the General Problem

The general idea of the solution behind this problem is easily explained: when serializing, write some sort of "class identifier" to the file. This can either be the name of the derived class or some sort of (user supplied) class identifier.
When deserializing again, the function first reads the class identifier and then calls the deserialization function for this particular class, which will now handle the correct deserialization of the object.
A very good starting point is the [extensive description from ISO CPP about serialization][ISOCPP].
They also have a section about this particular problem.

> Note:
> If you are by any chance interested in how the exact same problem can be nicely solved in Rust,
> have a look at [typetag][serde-typetag] from the developer of [serde][serde].

### How can a correct mapping between "class identifier" and the correct class be achieved?
Without loss of generality, we will assume the name of the derived class as a class identifier.
Usually, there will be a map from class name to class prototypes of the derived class or function pointers that will take the input stream and produce an object of the correct derived class.
I like the idea of function pointers more, because then you don't have "partly constructed objects only used to create a full object" laying around.
The signature of the deserialization function's pointer would look like `Shape* (*)(std::istream&)`.

For this, we can easily put a `static std::map<std::string, Shape* (*)(std::istream&)>` into the `Shape` class.
To handle things nicely, we can alter the functions in the `Shape` class:

```cpp
class Shape {
  // <snip>

  static void serialize(std::ostream& out, Shape* shape) {
    const std::string& class_id = shape->class_identifier();
    out.write(class_id.c_str(), class_id.size() + 1 /*terminating null*/);

    shape->serialize(out);  // Forward to the derived implementation.
  }

  static Shape* deserialize(std::istream& in) {
    std::string class_id;
    std::getline(in, class_id, '\0');

    auto deserialize_func = type_registry[class_id];
    Shape* derived_object = deserialize_func(in);
    return derived_object;
  }

  static std::map<std::string, Shape* (*)(std::istream&)> type_registry_;
};
```

Nice, this looks quite good now! The deserialization function of the derived class doesn't even need to know that it is part of this weird problem! 😄

While this is already quite nice, one big open question is: how do we populate the `type_registry_`?
Of course, we can do this per hand... okay, just kidding 😆

What we want is **self-registering types in a factory**!
This is a more general problem, but also applies to our current situation.

# Self-Registering Classes in a Factory

Luckily, there were people out there who [already looked][bfilipek] at [this problem][sacko87].
While my solution is largely based on the ideas of these two sources, I go one step further: solve some problems in the previous approaches and also do it header-only! Excited?

Okay, so the idea is to create a `static bool register_type(...)` function in the `Shape` class that can be called from the derived classes and adds one entry to the `type_registry`.

```cpp
class Shape {
  // <snip>

  using deserialize_func = Shape* (*)(std::istream&);

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

Nice, looks useful! But think about it: How does this even help? How should we call this function? In the constructor of the derived class? This won't really help, because then types are only registered when at least one object has been created before.

The idea is to call the `register_type()` function from a _static context_! Why? Because then it will be called when _static objects_ are initialized at program startup and this ensures that the function is called before any useful work is done! Nice! How can we do this? By creating a `static` field inside each derived class that gets assigned the returned value from `register_type()`.

More precisely, this would like this:
```cpp
class Rectangle : public Shape {
  static Shape* deserialize_rectangle(std::istream& in) {
      // ...
  }

  static const bool registered_;
};

// Usually we would do this in the .cpp file, but we wanted it to do header-only, right?
const bool Rectangle::registered_ = Shape::register_type("Rect", &deserialize_rectangle); 
```

When your program is started, one of the first steps is to initialize the static objects.

Thus, `registered_` will also be initialized by calling the `register_type()` function and our map in the `Shape` class is being populated, right?

Sadly, not necessarily. The chances that you end up with an uninitialized map are quite high.
The problem is that we ran into the [_"static initialization order fiasco"_][static-fiasco], because the order in which static values are initialized is undefined.
One of the [original posts][bfilipek] mentioned that this wouldn't happen because of "Zero initialization", but this sadly won't help us if the two classes are in different translations unit, which they probably are.

# Circumventing the Static Initialization Order Fiasco

But luckily, there is [a solution to this problem][fiasco-solution]:
If the classes are in different translation units, we can _force_ the initialization of the `type_registry_` but putting it inside a function call and call this function instead. How? By making use of `static` variables inside functions in our `Shape` class. This is also called the _Construct On First Use Idiom_.

```cpp
class Shape {
  // <snip>

 private:
  static std::map<std::string, Shape* (*)(std::istream&)>& get_type_registry() {
    // Static: One and the same instance for all function calls.
    static std::map<std::string, Shape* (*)(std::istream&)> type_registry;
    return type_registry;
  }
};
```

# Complete Solution

Nice! Now we even know how to deal with random static initialization order! Putting it all together, we end up with the following code:

```cpp
#include <iostream>
#include <string>
#include <map>

class Shape {
  // Function pointer that takes a stream and produces an object of a derived class.
  using deserialize_func = Shape* (*)(std::istream&);

 public:
  virtual ~Shape() = default;

  // Provide a high-level serialization function that stores the class_identifier.
  static void serialize(std::ostream& out, Shape* shape) {
    const std::string& class_id = shape->class_identifier();
    out.write(class_id.c_str(), class_id.size() + 1 /*terminating null*/);
    shape->serialize(out);
  }

  // Provide a high-level deserialization function
  // that dispatches to the correct deserialization function based
  // on class_identifier.
  static Shape* deserialize(std::istream& in) {
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

  // This function needs to be called by a derived class inside a *static* context.
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
  std::string class_identifier() const override {
    return "Rect";
  }

  void serialize(std::ostream& out) const override {
      // ...
  }

  static Shape* deserialize_rectangle(std::istream& in) {
      // ...
  }

 private:
  static const bool registered_;

  int64_t width_, height_;
};

// Statically register the "Rectangle" class to the factory.
const bool Rectangle::registered_ = Shape::register_type("Rect", &deserialize_rectangle);
```





[blockpartitioning]: https://github.com/andreaskipf/blockpartitioning
[ISOCPP]: https://isocpp.org/wiki/faq/serialization#serialize-inherit-no-ptrs
[factorypattern]: https://en.wikipedia.org/wiki/Factory_method_pattern
[serde]: https://github.com/serde-rs/serde
[serde-typetag]: https://github.com/dtolnay/typetag
[bfilipek]: https://www.bfilipek.com/2018/02/factory-selfregister.html
[sacko87]: https://gist.github.com/sacko87/3359911
[static-fiasco]: http://www.cs.technion.ac.il/users/yechiel/c++-faq/static-init-order.html
[fiasco-solution]: http://www.cs.technion.ac.il/users/yechiel/c++-faq/static-init-order-on-first-use.html
