---
layout: post
comments: true
title: Return value optimization (C++)
---
How many objects are created in the code below?

{% highlight cpp %}
class B;

B foo() {
  B x;
  return x;
}

int main() {
  B b = foo();
}
{% endhighlight %}

If you said **one** then you are right!

Let's try to figure out by implementing a very simple `B` class with a constructor, destructor and copy constructor.

{% highlight cpp %}
class B {
public:
  B() { cout << "Default constructor" << endl; }
  B(B const &b) { cout << "Copy constructor" << endl; }
  ~B() { cout << "Destructor" << endl; }
};
{% endhighlight %}

If we run the code we can see that the default constructor is called only once and no copy constructor is called at all. So what happened?

{% highlight cpp %}
Default constructor
Destructor
{% endhighlight %}

Although three objects should be created in theory (`x`, the returned object and `b`), most modern compilers will apply *return value optimization*.

Instead of creating the object `x` locally, the compiler creates that object where `foo` returns.

Should we run the same code without return value optimization (`-fno-elide-constructors` flag) the result would have been drastically different.

{% highlight cpp %}
Default constructor
Copy constructor
Destructor
Copy constructor
Destructor
Destructor
{% endhighlight %}
