---
categories: [cpp]
date: '2025-02-26T23:54:44-05:00'
layout: post
permalink: /c/class-definition-layout/
title: Class Definition Layout
---

When we begin reading a class definition, we read top-down. When we want to know how to use a class, we need to know how to construct it (via the constructor), then we look at the public member functions. Ideally, we never look at the private (or even protected) attributes: they are implementation details.

```cpp
class MyClass {
  public:
  void func1(int, char);
  void func2(long, int);
  void func3(int*);

  // Constructor
  MyClass() = default;
};
```

In the example above, the reader encounters member functions before they see the constructor. This is counter-intuitive. If the constructor was at the top, the caller would have a stronger idea of what this class was capable of, whether it’s default constructible or not, and what it’s dependencies (constructor arguments) are. Consider re-ordering it as follows:

```cpp
class MyClass {
  public:
  // Constructor
  MyClass() = default;
  
  void func1(int, char);
  void func2(long, int);
  void func3(int*);
};
```

Now, if someone is only interested in how to create the object and call `func1`, they need not bother skim through `func2` and `func3` to find all that they need.

This example did not show protected or private members, but the same logic applies. Since users of the class do not need to know what the protected or private members are, they should be declared at the bottom of the class definition.
