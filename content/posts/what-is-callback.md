---
title: "What Is a Callback? Why Do epoll, ThreadPool, and Async Programming Need It?"
date: 2026-07-14
draft: false
tags:
  - C++
  - Callback
  - Async
  - epoll
  - Event Driven
categories:
  - Many WHYs
---
In traditional programming, we usually think:

```text
Call Function

      |

      v

Get Result
```

Example:

```cpp
int result = calculate();
```

The caller controls the entire flow.

But modern systems often work differently.

Examples:

* epoll
* thread pools
* GUI frameworks
* asynchronous I/O
* networking servers

They all have one common problem:

> We do not know exactly when something will happen.

A network packet may arrive later.

A thread may finish later.

A file operation may complete later.

So instead of:

```text
Call -> Wait -> Return
```

we need:

```text
Register Interest

        |

        v

Something Happens

        |

        v

Execute Handler
```

This handler is called:

> Callback

---

# 1. Why Do Programs Need Callbacks?

A simple example explains the problem clearly.

A server may do this:

```cpp
read(socket);
```

The traditional approach:

```text
Thread

 |
 v

wait for data

 |
 v

process data
```

The thread is blocked.

If there are:

```
10,000 connections
```

we cannot create:

```
10,000 threads
```

because:

* memory overhead
* context switch cost
* scheduling overhead

---

Modern servers use:

```text
Event Driven Model
```

Instead:

```text
Register socket

        |

        v

Continue doing other work

        |

        v

When data arrives

        |

        v

Run callback
```

---

# 2. What If We Don't Know When an Event Happens?

This is the core problem.

Example:

A TCP server:

```cpp
accept();
```

When will a client connect?

Unknown.

Could be:

```
1 second later

10 minutes later

1 hour later
```

Should the program:

```cpp
while(true)
{
    check();
}
```

?

This is polling.

Example:

```cpp
while(true)
{
    if(data_ready())
    {
        handle();
    }
}
```

Problems:

* wastes CPU
* inefficient
* does not scale

---

The better model:

Tell the system:

> Notify me when something happens.

Example:

```cpp
epoll_wait();
```

The kernel waits.

When ready:

```
socket readable
```

The application receives:

```
event notification
```

Then:

```
callback()
```

runs.

---

# 3. What Is a Callback?

A callback is:

> A function passed to another component, which will be called later when a specific event happens.

Normal function:

```cpp
void handle()
{
    cout << "hello";
}


handle();
```

The programmer decides:

```
When to call
```

---

Callback:

```cpp
register_callback(handle);
```

The programmer says:

```
Here is the function.

Call it when needed.
```

The framework decides:

```
When to call
```

---

The relationship:

Normal call:

```text
Application

     |

     v

Function
```

Callback:

```text
Application

     |

register function

     |

     v

Framework

     |

event happens

     |

     v

Call function
```

---

# 4. Callback vs Normal Function Call

The difference is control flow.

## Normal Function Call

The caller controls execution.

Example:

```cpp
process_request();
```

Flow:

```text
main()

 |
 v

process_request()

 |
 v

return
```

---

## Callback

The caller gives control away.

Example:

```cpp
server.onMessage(handleMessage);
```

Flow:

```text
main()

 |
 v

register callback

 |
 v

continue running


(later)


event happens

 |
 v

handleMessage()
```

---

The key difference:

|                | Normal Call | Callback     |
| -------------- | ----------- | ------------ |
| Who calls?     | Caller      | Framework    |
| When executed? | Immediately | Later        |
| Control flow   | Synchronous | Event-driven |

---

# 5. How Is a Callback Implemented?

At the lowest level:

A callback is usually:

```
Function Address
```

The program stores:

```
where is the function?
```

---

Example:

```cpp
void hello()
{
    cout << "hello";
}


void register_callback(void (*cb)())
{
    cb();
}
```

The callback parameter:

```cpp
void (*cb)()
```

is a function pointer.

---

The memory looks like:

```
Function Pointer

       |

       v

Function Address

       |

       v

Machine Code
```

---

In C:

```c
void (*callback)(int);
```

means:

```
pointer to a function
```

---

C++ provides more flexible ways:

* function pointer
* std::function
* lambda
* functor

---

# 6. Why Are Lambdas Often Used With Callbacks?

Because callbacks usually need context.

Example:

A server receives a connection:

```cpp
onMessage(callback);
```

The callback may need:

* connection object
* user information
* server state

---

A normal function:

```cpp
void onMessage()
{
}
```

cannot easily capture outside variables.

---

Lambda solves this:

```cpp
server.setCallback(
    [this](Message msg)
    {
        handleMessage(msg);
    }
);
```

The lambda captures:

```
this
```

so it can access:

```
TcpServer

TcpConnection

EventLoop
```

---

Conceptually:

Lambda:

```cpp
[capture](parameters)
{
    body
}
```

creates an object containing:

```
Captured Data

+

operator()
```

---

Example:

```cpp
auto callback =
    [count](int x)
    {
        return count + x;
    };
```

The compiler creates something like:

```cpp
class LambdaObject
{
    int count;

public:

    int operator()(int x)
    {
        return count+x;
    }
};
```

---

# 7. Callback in epoll

Now connect everything.

Your server:

```text
socket()

     |

     v

fd

     |

     v

epoll_ctl()
```

Register interest:

```
Monitor fd
```

---

Event loop:

```cpp
while(true)
{
    events = epoll_wait();

    for(event : events)
    {
        callback(event);
    }
}
```

---

The flow:

```text
Client sends data

        |

        v

NIC receives packet

        |

        v

Kernel TCP Stack

        |

        v

Socket buffer

        |

        v

epoll marks fd ready

        |

        v

epoll_wait returns

        |

        v

Read Callback executes
```

---

# 8. Callback in Thread Pool

Thread pool has the same idea.

Instead of:

```cpp
create thread

execute task
```

we do:

```cpp
submit(task_callback)
```

Example:

```cpp
pool.submit(
    [](){
        doWork();
    }
);
```

The thread pool decides:

```
Which thread runs it?

When does it run?
```

---

Flow:

```text
Task submitted

       |

       v

Queue

       |

       v

Worker thread

       |

       v

Callback execution
```

---

# 9. Callback Is the Foundation of Event-Driven Systems

Many modern systems use this pattern:

```
Event

 |

 v

Notification

 |

 v

Callback

 |

 v

Handler
```

Examples:

## Networking

```
epoll

 |

read callback
```

## GUI

```
button click

 |

onClick()
```

## Async IO

```
operation complete

 |

completion callback
```

## Thread Pool

```
task ready

 |

worker callback
```

---

# Final Knowledge Chain

Why do systems need callbacks?

```text
Events happen unpredictably

        |

        v

Cannot block waiting

        |

        v

Register interest first

        |

        v

System notifies later

        |

        v

Execute callback

        |

        v

Build asynchronous systems
```

---

# Conclusion

Callback is not just a programming technique.

It is a control-flow model.

Traditional programming:

```
I call you.
```

Callback programming:

```
I tell you what to call.

You call me when something happens.
```

This idea is the foundation of:

* epoll-based servers
* Reactor pattern
* asynchronous programming
* thread pools
* modern C++ network frameworks

Understanding callbacks is the bridge from:

```
Linux Kernel Events

        |

        v

Event Loop

        |

        v

High Performance Server Design
```
