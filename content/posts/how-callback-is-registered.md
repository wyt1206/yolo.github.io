---
title: "How Are Callbacks Registered? From Function Registration to Event-Driven Systems"
date: 2026-07-16
draft: false
tags:
  - C++
  - Callback
  - Lambda
  - Event Driven
  - Design Pattern
categories:
  - Many WHYs
---
In the previous article, we learned:

> A callback is a function that is called later when an event happens.

But a deeper question appears:

If the system calls our function later:

> How does the system remember which function to call?

For example:

```cpp
server.onMessage(handleMessage);
```

After this line executes:

Where is:

```
handleMessage
```

stored?

How does the server find it later?

This leads to the next question:

```
Callback

    |

    v

Registration

    |

    v

Storage

    |

    v

Invocation
```

---

# 1. Why Do We Need to Register a Callback?

Because the event producer and event consumer are separated.

Example:

A network server:

```
Client

   |

   v

Socket

   |

   v

Server
```

The server does not know:

```
When will data arrive?
```

It could be:

```
1 ms later

10 seconds later

1 hour later
```

So the server cannot simply:

```cpp
read();
handle();
```

Instead:

First:

```cpp
register callback
```

Meaning:

> When this event happens, call this function.

---

The flow becomes:

```
Register

   |

   v

Wait

   |

   v

Event happens

   |

   v

Execute callback
```

---

# 2. What Does Callback Registration Actually Do?

Consider:

```cpp
eventLoop.setCallback(handleRead);
```

Many people imagine:

```
Magic happens
```

But internally it is usually:

```
Store something
```

The system saves:

```
Which function?

With what data?
```

---

The simplest implementation:

```cpp
using Callback = void(*)();


Callback callback;


void registerCallback(Callback cb)
{
    callback = cb;
}
```

After:

```cpp
registerCallback(myFunction);
```

Memory:

```
callback variable

        |

        v

address of myFunction
```

---

Later:

```cpp
callback();
```

The program jumps to:

```
myFunction()
```

---

So callback registration is fundamentally:

> Save executable code information for later execution.

---

# 3. Is Callback Registration Only Function Pointer and Lambda?

No.

Function pointer is only the simplest form.

There are several common approaches.

---

## 3.1 Function Pointer

C style:

```cpp
void onEvent()
{
}


registerCallback(onEvent);
```

Internally:

```
function address
```

---

Advantages:

* simple
* low overhead

Limitations:

Cannot easily store state.

---

# 3.2 Functor Object

C++ allows objects to behave like functions.

Example:

```cpp
class Handler
{
public:

    void operator()()
    {
        handle();
    }
};
```

Usage:

```cpp
Handler h;

callback = h;
```

The object contains:

```
data

+

operator()
```

---

# 3.3 std::function

Modern C++ often uses:

```cpp
std::function<void()> callback;
```

It can store:

* function pointer
* lambda
* functor
* member function wrapper

Example:

```cpp
callback = []()
{
    cout << "event";
};
```

---

# 3.4 Lambda

Lambda is not a special callback mechanism.

It is another way to create a callable object.

Example:

```cpp
auto callback =
    [this]()
    {
        handleRead();
    };
```

The compiler creates something similar to:

```cpp
class LambdaObject
{
    Server* thisPtr;

public:

    void operator()()
    {
        thisPtr->handleRead();
    }
};
```

---

# 4. Does Callback Registration Always Save a Function Address?

Not exactly.

This is a common misunderstanding.

A callback registration stores:

> Something callable.

It could be:

```
Function address
```

or:

```
Object containing function logic
```

or:

```
Lambda object
```

or:

```
std::function wrapper
```

---

Example:

Lambda:

```cpp
[this]()
{
    read();
}
```

The stored object may contain:

```
captured variables

+

operator()
```

Not just an address.

---

The real model:

```
Callback Storage

        |

        +----------------+

        |                |

 Function pointer    Callable object

                        |

                        v

                    Lambda/Functor
```

---

# 5. Is epoll Callback Registration the Same as C++ Lambda Registration?

Important question.

The answer:

> No.

They are related concepts but different layers.

---

C++ callback:

Example:

```cpp
channel.setReadCallback(
    [](){
        readData();
    }
);
```

This happens in user space.

The program stores:

```
callable object
```

---

epoll registration:

Example:

```cpp
epoll_ctl(
    epfd,
    EPOLL_CTL_ADD,
    fd,
    &event
);
```

This happens across:

```
User Space

        |

        v

Kernel Space
```

---

epoll stores:

```
fd

interest events

user data
```

Inside kernel:

```
epoll instance

      |

      v

interest list

      |

      v

file descriptor
```

---

When an event happens:

```
socket ready

      |

      v

epoll_wait returns fd

      |

      v

user code finds callback

      |

      v

execute lambda
```

---

The complete flow:

```
Kernel:

socket ready

    |

    v

epoll_wait returns


User Space:

fd

    |

    v

Channel

    |

    v

callback()

```

---

# 6. Callback Registration in an Event Loop

A typical Reactor server:

```cpp
Channel channel(fd);


channel.setReadCallback(
    [this]()
    {
        handleRead();
    }
);
```

What happens?

Step 1:

Store callback:

```
Channel

 |
 +-- readCallback
```

---

Step 2:

Register fd:

```cpp
epoll_ctl()
```

Kernel remembers:

```
fd is interesting
```

---

Step 3:

Event happens:

```
socket readable
```

---

Step 4:

EventLoop:

```cpp
while(true)
{
    events = epoll_wait();

    for(auto event: events)
    {
        event.callback();
    }
}
```

---

# 7. Why Is Callback Registration the Core of Event-Driven Systems?

Because event-driven systems separate:

```
Detection

and

Handling
```

---

Without callbacks:

```text
Wait

Check

Handle

Wait

Check

Handle
```

Polling.

---

With callbacks:

```text
Register interest

        |

        v

Do other work

        |

        v

Receive notification

        |

        v

Execute handler
```

---

This creates:

* scalability
* loose coupling
* asynchronous execution

---

# 8. Relationship Between Callback and Observer Pattern

They are closely related.

Observer Pattern:

```
Subject

   |

   +---- Observer A

   |

   +---- Observer B
```

The subject notifies observers when state changes.

---

Example:

```cpp
button.onClick(observer);
```

When clicked:

```cpp
observer.update();
```

---

Callback:

```
Event

   |

   v

Function
```

Observer:

```
Event

   |

   v

Multiple subscribers
```

---

Relationship:

```
Callback

    |

    v

One notification mechanism


Observer Pattern

    |

    v

Structured callback system with multiple listeners
```

---

# 9. Main Applications of Callbacks

## Networking

Examples:

* epoll
* Node.js
* Netty
* Boost.Asio

Flow:

```
socket event

      |

      v

callback

      |

      v

handle request
```

---

## Thread Pool

Example:

```cpp
pool.submit(
    [](){
        work();
    }
);
```

Flow:

```
task ready

    |

    v

worker executes callback
```

---

## GUI Programming

Example:

```cpp
button.onClick(callback);
```

---

## Async Programming

Example:

```
request sent

      |

      v

response arrives

      |

      v

completion callback
```

---

# Final Knowledge Chain

How does callback registration work?

```
Need asynchronous execution

        |

        v

Register interest

        |

        v

Store callable object

        |

        v

Event happens

        |

        v

Find stored callback

        |

        v

Execute handler
```

---

# Conclusion

Callback registration is the bridge between:

```
Unknown Future Events

        |

        v

Program Execution
```

The core idea is simple:

> Tell the system what to execute later.

This idea powers:

* epoll servers
* Reactor pattern
* thread pools
* async frameworks
* modern C++ networking

Understanding callback registration means understanding how modern event-driven systems are built.

````

---

加入这一篇以后，你的 ManyWhys 会形成非常漂亮的 Event-Driven 链：

```text
第8篇 User Mode / Kernel Mode
             |
             v
第9篇 epoll (下一篇)
             |
             v
第10篇 What is Callback?
             |
             v
第11篇 How Callback is Registered?
             |
             v
Reactor Pattern
             |
             v
C++ EventLoop Server
````
