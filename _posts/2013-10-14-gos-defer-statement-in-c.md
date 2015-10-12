---
layout: post
title: Go’s defer statement in C++
---

The Go language provides a useful [defer statement](http://blog.golang.org/defer-panic-and-recover) to guarantee certain code is always executed when returning from the current scope. Though we can use constructor in C++, things get tricky e.g. when a pointer needs to be deleted. Here we present some simple draft code to solve this issue.

{% highlight cpp %}
#include <functional>
 
// the user should fill copy and move constructors
class DeferredAction
{
public:
    DeferredAction(const std::function<void(void)>& deferred) : m_deferred(deferred) {}
 
    ~DeferredAction() {
        if (m_deferred)
            m_deferred();
    }
 
private:
    std::function<void(void)> m_deferred;
};
{% endhighlight %}

Then if the client code looks like the following:

{% highlight cpp %}
#include "deferredaction.h"
 
#include <iostream>
 
int main()
{
    DeferredAction deferred([] () {
        std::cout << "executed when the 'deferred' object is destructed" << std::endl;
    });
    std::cout << "print some random stuff here" << std::endl;
    return 0;
}
{% endhighlight %}

It will first print the line: *print some random stuff here*. Then when the deferred object is destructed when main() returns, it prints the second line: *executed when the 'deferred' object is destructed*.
