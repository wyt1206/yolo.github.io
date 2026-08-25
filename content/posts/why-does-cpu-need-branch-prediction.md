---
title: "Why Does CPU Need Branch Prediction?"
date: 2026-08-25
draft: false
tags:
  - CPU
  - Computer Architecture
  - Branch Prediction
  - Performance
  - Low Latency
categories:
  - ManyWhys
---
Consider a simple piece of code:

```cpp
if (x > 0) {
    processA();
}
else {
    processB();
}
```

For a programmer, this looks simple.

The CPU needs to know:

```
Is x > 0?
```

before deciding whether to execute:

```
processA()

or

processB()
```

A natural solution would be:

```
calculate condition

↓

wait for result

↓

fetch next instruction
```

But modern CPUs do not work this way.

Because waiting is expensive.

This leads to a question:

> If the CPU does not know the answer yet, why doesn't it just guess?

This is where:

```
Branch Prediction
```

comes in.

---

# Why 1: Why does CPU need Branch Prediction?

Modern CPUs use instruction pipelines.

A simplified pipeline looks like:

```
Fetch

↓

Decode

↓

Execute

↓

Memory

↓

Write Back
```

The goal is:

> Keep every stage busy every cycle.

For example:

```
Cycle 1:

Instruction 1 → Execute

Instruction 2 → Decode

Instruction 3 → Fetch
```

Multiple instructions are processed at the same time.

---

But a branch instruction creates a problem.

For example:

```cpp
if(condition)
{
    ...
}
```

The CPU does not immediately know:

```
Which instruction should come next?
```

Should it fetch:

```
processA()
```

or:

```
processB()
```

?

---

If the CPU waits:

```
wait for condition

↓

know the result

↓

continue fetching
```

the pipeline becomes empty.

This is called:

```
Pipeline Stall
```

The CPU loses cycles doing nothing.

---

So the CPU asks:

> Can I predict the future before I know the answer?

That is the idea behind Branch Prediction.

---

# Why 2: How does CPU predict a branch?

The CPU does not understand program logic.

It does not know:

```
this is a loop

this is an if statement

this function is important
```

Instead, it relies on:

```
Execution History
```

---

Because programs usually have patterns.

Example:

```cpp
for(int i = 0; i < 1000; i++)
{

}
```

The branch behavior is:

```
Taken

Taken

Taken

...

Taken

Not Taken
```

Most of the time:

```
Taken
```

The CPU learns:

> This branch usually goes the same way.

So next time:

```
Predict: Taken
```

---

# Why 3: How does CPU remember previous branches?

The CPU contains dedicated hardware for this.

One important component is:

```
Branch History Register (BHR)
```

It records previous branch outcomes.

For example:

```
Taken
Taken
Not Taken
Taken
```

Can be represented as:

```
1101
```

Where:

```
1 = Taken

0 = Not Taken
```

---

Important:

BHR is not like:

```
RAX

RBX

RCX
```

These are programmer-visible registers.

BHR is hidden CPU state.

A simplified view:

```
CPU Core

 └── Front End

       └── Branch Predictor

              ├── BHR
              ├── PHT
              └── BTB
```

---

# Why 4: Is history enough?

Not always.

The CPU needs a way to convert history into prediction.

This is where:

```
Pattern History Table (PHT)
```

comes in.

---

Example:

The CPU sees:

```
BHR = 1011
```

It looks up:

```
PHT

1011 → Taken

1010 → Not Taken
```

Then it predicts:

```
Next branch = Taken
```

---

But there is a problem.

What if one prediction is wrong?

Should the CPU immediately change its mind?

No.

Programs usually have stable behavior.

One unusual event should not completely change the prediction.

This leads to:

```
2-bit Saturating Counter
```

---

# Why 5: How does 2-bit Saturating Counter work?

Each branch can have four states:

```
Strongly Taken

        ↓

Weakly Taken

        ↓

Weakly Not Taken

        ↓

Strongly Not Taken
```

---

Example:

Current state:

```
Strongly Taken
```

Prediction:

```
Taken
```

Actual result:

```
Taken
```

The state remains:

```
Strongly Taken
```

---

Now:

Prediction:

```
Taken
```

Actual result:

```
Not Taken
```

The CPU does not immediately switch.

Instead:

```
Strongly Taken

        ↓

Weakly Taken
```

---

Why?

Because program behavior is usually predictable.

The CPU avoids changing prediction because of one abnormal case.

---

# Why 6: How does CPU know where to jump?

Branch prediction has two questions:

## Question 1:

```
Will the branch be taken?
```

Handled by:

```
Branch Predictor
```

---

## Question 2:

```
Where should execution continue?
```

Handled by:

```
Branch Target Buffer (BTB)
```

---

BTB stores:

```
Branch Address

        ↓

Target Address
```

Example:

Previously:

```
0x1000

↓

0x5000
```

The next time CPU sees:

```
0x1000
```

it can immediately fetch instructions from:

```
0x5000
```

without waiting.

---

# Why 7: What happens if CPU predicts incorrectly?

The CPU cannot simply continue.

It has already executed instructions from the wrong path.

Example:

CPU predicts:

```
Path A
```

Actually:

```
Path B
```

The CPU must discard:

```
A1

A2

A3
```

and restart from:

```
Path B
```

---

This is called:

```
Pipeline Flush
```

The lost time is:

```
Branch Misprediction Penalty
```

---

# Why 8: Why does Branch Prediction matter for Low Latency Systems?

For normal applications:

One wrong prediction may not matter.

But for:

* HFT systems
* network processing
* operating systems
* high-performance C++

every cycle matters.

---

Performance is not only:

```
Algorithm Complexity
```

It also depends on:

```
Cache Miss

+

Memory Access

+

Branch Misprediction

+

Pipeline Stall
```

---

A theoretically efficient algorithm can still be slow if it constantly causes CPU stalls.

---

# Connection With Low Latency Engineering

Low latency engineers often ask:

```
Where is the data?

How does CPU access it?

How predictable is execution?
```

Because modern performance depends on the interaction between:

```
Software

    ↓

Compiler

    ↓

CPU Architecture

    ↓

Hardware Execution
```

---

# Final Thought

A programmer sees:

```cpp
if(condition)
```

The CPU sees:

```
Branch

↓

History

↓

Prediction

↓

Speculative Execution

↓

Rollback (if wrong)
```

Branch Prediction is not about the CPU knowing the future.

It is about:

> Using past execution patterns to make a good guess, allowing the CPU to continue working before the final answer is available.

```
History

↓

Prediction

↓

Speculative Execution

↓

Higher Performance
```
