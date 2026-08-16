---
title: "Why Does epoll Need Both a Red-Black Tree and a Ready List?"
date: 2026-08-16
draft: false
tags:
  - Linux
  - epoll
  - Kernel
  - Networking
categories:
  - Linux
---
When looking at how `epoll` works internally, two data structures often come up:

- a red-black tree
- a ready list

Why does `epoll` need both?

Could one data structure do the job?

---

## Q1: What does epoll actually need to keep track of?

There are two different things to keep track of.

First:

> Which file descriptors are being monitored?

Second:

> Which of those file descriptors currently have events to report?

For example, an epoll instance might be watching:

```text
1,000,000 file descriptors
```

while only:

```text
100 file descriptors
```

are currently ready.

These are not the same set.

```text
Monitored fds
    ↓
What am I watching?

Ready fds
    ↓
What needs attention now?
```

That difference is the starting point for understanding the two data structures.

---

## Q2: Why can't epoll use the same structure for both?

Because the operations are different.

For the monitored file descriptors, epoll needs to support operations such as:

```text
ADD
MOD
DEL
LOOKUP
```

For ready events, the access pattern is different:

```text
event occurs
    ↓
mark the fd as ready
    ↓
epoll_wait()
    ↓
consume the event
```

One is mainly about managing objects.

The other is about delivering events.

Trying to make one structure handle both jobs usually means making one of the operations unnecessarily expensive.

---

## Q3: Why is a red-black tree useful for the monitored fds?

Consider:

```c
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &event);
```

The kernel needs to associate the fd with its epoll state.

Later, the application might call:

```c
epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &event);
```

or:

```c
epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
```

So the kernel needs a way to find the object associated with a particular fd.

That's a lookup problem.

A balanced tree works well for this kind of operation.

Conceptually:

```text
             epoll
               |
        interest structure
               |
        red-black tree
        /      |      \
      fd      fd      fd
```

The tree is mainly concerned with:

> Which file descriptors belong to this epoll instance?

---

## Q4: Then what problem does the ready list solve?

Now consider the other side.

Suppose the epoll instance monitors one million file descriptors.

Only a small number of them may currently have events.

When `epoll_wait()` is called, we don't want to search through the entire monitored set again.

We already know which objects became ready.

So the ready list keeps track of those objects:

```text
ready list

fd 37
  ↓
fd 892
  ↓
fd 12004
  ↓
...
```

The idea is simple:

> Don't search for events that the kernel already knows about.

---

## Q5: Why not use the red-black tree for ready events too?

Because `epoll_wait()` is not trying to find a particular fd.

It wants to retrieve events that are already known to be ready.

The access pattern looks more like:

```text
event occurs
    ↓
add to ready list
    ↓
epoll_wait()
    ↓
get ready events
```

This is closer to consuming a queue than searching a tree.

The red-black tree is useful when the question is:

> Where is fd 42?

The ready list is useful when the question is:

> What is ready right now?

Those are different questions.

---

## Q6: Why not use only a ready list?

Because epoll still needs to manage the complete set of monitored file descriptors.

Imagine:

```text
fd1 → fd2 → fd3 → ... → fd1,000,000
```

Now suppose the application wants to modify or remove one particular fd.

With only a simple list, finding that fd could require walking through a large number of entries.

That's not what we want for operations such as:

```text
epoll_ctl(ADD)
epoll_ctl(MOD)
epoll_ctl(DEL)
```

So a separate lookup structure makes sense.

---

## Q7: Does a file descriptor move from the tree to the ready list?

No.

This is an important distinction.

The fd remains part of the monitored set.

When an event occurs, the corresponding epoll object can also be placed on the ready list.

Conceptually:

```text
             monitored set
                  |
                fd 42
               /     \
              ↓       ↓
      red-black tree  ready list
```

The two structures represent two different views of the same monitored object.

The fd doesn't stop being monitored just because its current event has been consumed.

---

## Q8: What happens when fd 42 becomes readable?

At a high level:

```text
fd 42
  |
  | becomes readable
  ↓
epoll event handling
  |
  ↓
ready list
  |
  ↓
epoll_wait()
  |
  ↓
userspace receives the event
```

The important part here is not the wake-up mechanism itself.

The important part is the separation:

```text
red-black tree
    ↓
keeps track of the monitored object

ready list
    ↓
keeps track of its current ready state
```

---

## Q9: Why is separating these two states useful?

Because the lifetime of a monitored fd and the lifetime of an event are different.

A socket can be monitored for a long time:

```text
ADD
 ↓
monitor
 ↓
event
 ↓
event consumed
 ↓
monitor again
 ↓
event
 ↓
...
 ↓
DEL
```

Consuming an event does not mean removing the fd from epoll.

So:

```text
Monitoring lifecycle
    ≠
Event lifecycle
```

The data structures reflect that difference.

---

## Q10: Is epoll fast simply because it uses a red-black tree?

No.

That's an easy conclusion to draw, but it's incomplete.

The red-black tree helps with managing the set of monitored file descriptors.

The ready list helps with handling the file descriptors that currently have events.

They solve different problems:

```text
Red-black tree
    ↓
manage monitored objects

Ready list
    ↓
deliver ready events
```

The interesting part of epoll is not simply:

> "It uses a red-black tree."

It's the fact that epoll separates:

> managing what is being monitored

from:

> delivering what is currently ready.

---

## The whole picture

```text
                    epoll
                      |
             ┌────────┴────────┐
             ↓                 ↓
      Monitored objects    Ready events
             ↓                 ↓
      Red-black tree        Ready list
             ↓                 ↓
       "What am I          "What's ready
        watching?"            now?"
             |                 |
             └────────┬────────┘
                      ↓
                 epoll_wait()
                      ↓
                userspace events
```

The red-black tree and the ready list are not two copies of the same information.

They exist because epoll has two different jobs to do:

> **manage the set of monitored file descriptors, and deliver the ones that are currently ready.**

That's the reason epoll needs both.

---
