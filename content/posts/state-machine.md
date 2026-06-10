---

title: "I Didn't Understand State Machines Until I Asked These 5 Questions"

date: 2026-06-10

draft: false

---

Recently, I spent some time learning about State Machines (FSMs).

At first, every explanation sounded the same:

✅ State

✅ Event

✅ Transition

But I kept asking myself:

Isn’t this just a state transition flow?

After digging deeper, I finally understood what makes State Machines important.

⸻

1. What is a State Machine?

In one sentence:

Current State + Event = Next State

For example, in an order system:

Pending Payment → Paid → Shipped → Delivered

When:

Current State = Pending Payment

Event = Payment Success

Result:

State becomes Paid

Simple enough.

⸻

2. Why is it called a “State Machine”?

My first thought was:

Isn’t this just a flowchart?

Not exactly.

A flowchart describes:

“What’s the next step?”

A state machine describes:

“What’s the current state, and what transitions are allowed?”

Example:

Pending Payment
├─ Payment Success → Paid
└─ Cancel → Cancelled

The focus isn’t the process.

The focus is:

Only legal state transitions are allowed.

That’s why it’s called a State Machine, not a State Diagram.

⸻

3. But Flowcharts Can Have Conditions Too!

This was the question that confused me the most.

Eventually I realized:

Flowcharts focus on:

👉 How tasks are executed

State Machines focus on:

👉 How states change

Flowchart:

Start
↓
Decision
↓
Action A
↓
Action B

State Machine:

Pending Payment
↓ Payment Success
Paid
↓ Ship
Shipped

Flowcharts manage execution paths.

State Machines manage system states.

⸻

4. What’s the Difference Between a State Machine and if-else?

Here’s the surprising part:

A State Machine is often implemented using if-else statements.

Example:

if state == “Pending Payment” and event == “Payment Success”:
state = “Paid”

So what’s the difference?

if-else:

Rules are scattered across code.

State Machine:

Rules are organized into a model.

The logic becomes visible and easier to reason about.

⸻

5. Why Do We Need State Machines Then?

Because complexity grows faster than we think.

Imagine a game character:

Standing
Running
Jumping
Attacking
Stunned
Dead

And events such as:

Press Attack
Press Jump
Take Damage
Health Reaches Zero

Soon you’ll start asking:

Can a dead character attack?

Can a stunned character jump?

What happens if damage is received while jumping?

At that point, you’re no longer writing features.

You’re maintaining a huge transition table.

That’s where State Machines become valuable.

⸻

6. Why Didn’t State Machines Feel Important to Me?

Because my systems weren’t complex enough.

If you have:

3 states

5 events

That’s only 15 combinations.

Your brain can handle it.

Simple if-else logic works perfectly fine.

But what about:

20 states

30 events

Potential combinations:

20 × 30 = 600

Now things become difficult to reason about.

This is where State Machines start to shine.

⸻

7. Do Large Systems Use One Huge State Machine?

Actually, no.

Because State Machines can suffer from something called:

State Explosion

For example:

Movement States:

Standing
Running
Jumping

Weapon States:

Unarmed
Armed
Reloading

If you combine everything into one machine:

Standing + Unarmed
Standing + Armed
Running + Armed
Jumping + Reloading
…

The number of states explodes.

⸻

8. How Do Real Systems Solve This?

They split the problem.

Instead of one giant state machine:

MovementFSM

WeaponFSM

HealthFSM

Each machine manages one concern.

For example:

Movement = Running

Weapon = Armed

Health = Healthy

Together they describe the full situation.

This is much easier to maintain.

⸻

9. My Final Understanding

State Machines are not necessary for making software run.

Software can work perfectly fine without them.

The real value is:

Helping humans understand and manage complex behavior.

Small systems:

if-else is enough.

Medium systems:

Finite State Machines (FSMs).

Large systems:

Hierarchical State Machines + Multiple FSMs.

Enterprise systems:

Workflow Engines.

⸻

Final Takeaway

State Machines are not for computers.

Computers can execute if-else just fine.

State Machines are for developers.

They help answer three critical questions:

• Where is the system right now?

• What is allowed to happen next?

• Which transitions are valid?

That’s why State Machines become increasingly valuable as systems grow in complexity.