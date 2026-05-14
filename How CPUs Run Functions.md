
# How CPUs Execute Functions: Function Calls, Stack Mechanics, Inlining vs Outlining – Master Notes

---

# 1. Revision Key Sentences

## 1.1 Core Problem: How Can a CPU Execute Functions?

- Every running program constantly stops what it is doing, goes somewhere else to perform a small task, and then continues exactly where it left off.
- CPUs fundamentally execute instructions **sequentially**, so function calls are not inherently built-in concepts—they must be implemented using lower-level control flow mechanisms.
- From the hardware's perspective, there is nothing inherently special about calling functions.
- The **compiler** is responsible for generating machine instructions that make function execution possible.
- The compiler translates high-level source code into a precise sequence of machine instructions that the processor executes.
- Assembly language is used to explain this because it is a human-readable textual representation of machine code.

---

## 1.2 Two Compiler Strategies for Function Calls

### Function Inlining

- Function inlining copies the entire function body into the caller at every call site.
- Variables are resolved separately for each copied instance.
- The resulting executable behaves as though the code was written as one long instruction sequence.
- CPU execution remains purely sequential.
- Compilers attempt to inline functions whenever possible.

---

### Function Outlining

- Function outlining emits function code only once.
- Function code is stored in a separate memory region.
- When called, execution jumps to that function’s location.
- After completion, execution must return to the original caller location.
- Additional compiler-generated instructions are required for:
  - Jumping into the function
  - Returning to the caller

---

## 1.3 Jump Instructions as the Fundamental Mechanism

- Function calls can be implemented using ordinary **jump instructions**.
- A jump tells the CPU:
  
  **Do not execute the next instruction; continue at another address instead.**

- Instruction addresses are often written symbolically as **labels** in assembly.
- Labels are resolved into concrete memory addresses during executable generation.
- Jump instructions are fundamental for:
  - Function calls
  - Loops
  - Conditional branching

---

## 1.4 Naive Function Return Strategy: Hardcoded Return Label

- Caller jumps to function code.
- Function ends with a jump back to a hardcoded return label.
- This only works if the function is called from exactly one location.

### Problem

- Functions can be called from multiple places.
- Hardcoded return addresses force all calls to return to the same location.
- This makes general-purpose functions impossible.

---

## 1.5 Dynamic Return Address Variable

### Basic Idea

- Before calling a function, store the desired return address in a variable:

\[
\text{return\_address} = \text{address after call}
\]

- Then jump into the function.
- At function end:
  - Read `return_address`
  - Jump there

---

### Works for Sequential Calls

- Main stores return address.
- Function executes.
- Function reads return address and returns correctly.
- Repeated calls work as long as calls are not nested.

---

### Failure with Nested Calls

Example:

```text
main → function A → function B
```

- `main` writes its return address.
- `A` overwrites it when calling `B`.
- `B` returns correctly.
- `A` now has lost its own return address.
- Return goes to the wrong place.
- Program may loop forever.

---

## 1.6 Need for a Dynamic Data Structure

- Separate variables for every call are not scalable.
- Dynamic problems require dynamic solutions.
- The correct data structure is a **stack**.

---

## 1.7 Stack Fundamentals

- Stack stores elements **on top of each other**.
- Uses **Last-In, First-Out (LIFO)** behavior.

### Operations

#### Push

Add newest element.

#### Pop

Remove most recently added element.

---

## 1.8 Implementing a Stack in Assembly

### Memory Allocation

- Reserve a memory region for the stack.
- Example:

\[
\text{Stack starts at address } 9999
\]

- Stack grows **downward** toward lower memory addresses.

---

### Stack Pointer

- Variable storing address of top stack element.

\[
SP = \text{top of stack}
\]

---

### Push Algorithm

1. Read stack pointer.
2. Decrement it:

\[
SP = SP - 1
\]

3. Store value at new address.
4. Update stack pointer.

---

### Pop Algorithm

1. Read stack pointer.
2. Read value stored at that location.
3. Increment stack pointer:

\[
SP = SP + 1
\]

4. Update pointer.

---

## 1.9 Using the Stack for Function Returns

### On Function Call

- Push return address.
- Jump to function.

---

### On Function Return

- Pop return address.
- Jump to popped address.

---

### Nested Calls Now Work

Example:

```text
main
 └── A
      └── B
```

Execution:

- Push return-to-main
- Jump to A
- Push return-to-A
- Jump to B
- B pops return-to-A
- Return to A
- A pops return-to-main
- Return to main

---

## 1.10 Compiler Must Inline Stack Logic Initially

- Reusable push/pop helper functions cannot be used yet.
- That would require function calling before function calling exists.
- Therefore compiler must inject stack manipulation code directly at each call site.

---

## 1.11 Hardware Support for the Stack

CPUs eventually added hardware support.

### Dedicated Stack Pointer Register

Instead of:

```text
stack_pointer stored in memory
```

Use:

```text
stack_pointer stored in CPU register
```

Benefits:

- Faster access
- Avoids two memory operations
- Improves performance

---

## 1.12 Dedicated Stack Instructions

### PUSH

Performs:

\[
SP = SP - 1
\]

Store value.

All in one instruction.

---

### POP

Reads value.

Then:

\[
SP = SP + 1
\]

All in one instruction.

---

## 1.13 CALL and RET Instructions

These are simply shortcuts built on top of stack operations.

### CALL

Equivalent to:

```text
push(return_address)
jump(function)
```

---

### RET

Equivalent to:

```text
address = pop()
jump(address)
```

---

## 1.14 Function Parameters and Return Values

Compiler must inject additional logic.

---

### Passing via Registers

Caller writes arguments into designated CPU registers.

Function reads from those registers directly.

Example concept:

```text
R1 = arg1
R2 = arg2
CALL foo
```

Return values can also be placed in registers.

---

### Passing via Memory

Caller writes arguments to memory.

Function loads them when needed.

Return values can also be written to memory.

---

### Why Memory Passing is Needed

- Some arguments are too large for registers.
- Number of arguments may exceed available registers.

---

### Calling Convention

Rules for:

- Which registers hold arguments
- Which registers hold return values
- Which memory locations are used

Defined by:

**ABI (Application Binary Interface)**

---

## 1.15 Performance Costs of Outlining

Function outlining adds overhead:

- Push return address
- Jump
- Execute function
- Pop return address
- Jump back

Additional argument handling may also be required.

---

## 1.16 Why Inlining Can Be Better

Benefits:

- No jumps
- No stack manipulation
- No parameter transfer overhead
- Fewer executed instructions
- Better locality

---

## 1.17 Why Inlining Can Be Worse

If large function is called many times:

- Function body duplicated repeatedly
- Executable size grows dramatically

This is **code bloat**.

---

## 1.18 Situational Tradeoff

Compiler must decide based on:

- Function size
- Call frequency
- Cache behavior
- Purpose of function

---

## 1.19 Cache Effects

### Inlining May Improve Cache Locality

If function is inside loop:

- Outlined version may cause instruction cache misses.
- CPU stalls waiting for instruction cache refill.

Inlining may keep caller and callee together in cache.

---

### Outlining May Improve Cache Efficiency

Rarely executed error-handling functions:

- If outlined, they stay outside hot code path.
- Frequently executed code can fit entirely in cache.

---

## 1.20 Compiler Hints

### C

```c
inline int foo() { ... }
```

---

### Rust

```rust
#[inline]
fn foo() { ... }
```

These suggest—but do not guarantee—inlining.

---

## 1.21 Recursion Cannot Be Inlined

If recursion depth is unknown at compile time:

\[
\text{Impossible to duplicate infinite call levels}
\]

Therefore outlining is essential.

---

## 1.22 Recursive Variable Overwrite Problem

If parameter memory is hardcoded:

```text
factorial(n):
   write n to memory
   factorial(n-1)
```

Each recursive call overwrites previous call’s variable.

Program becomes incorrect.

---

## 1.23 Stack Solves Recursion Too

Runtime stack can store:

- Return addresses
- Local variables
- Function parameters

This prevents recursive calls from overwriting earlier call frames.

---

# 2. Key Concepts, Definitions & Formulas

| Concept | Definition |
|---|---|
| **Compiler** | Program that translates source code into machine instructions |
| **Assembly** | Human-readable representation of machine code |
| **Function Inlining** | Copying function body into caller |
| **Function Outlining** | Emitting function separately and jumping to it |
| **Jump Instruction** | Instruction that changes execution address |
| **Return Address** | Address execution should resume after function |
| **Stack** | LIFO data structure |
| **Stack Pointer (SP)** | Address of top stack element |
| **PUSH** | Add value to stack |
| **POP** | Remove value from stack |
| **CALL** | Push return address + jump |
| **RET** | Pop return address + jump |
| **ABI** | Calling convention specification |

---

### Core Stack Equations

#### Push

\[
SP = SP - 1
\]

```text
MEM[SP] = value
```

---

#### Pop

```text
value = MEM[SP]
```

\[
SP = SP + 1
\]

---

### CALL Expansion

```text
push(return_address)
jump(function_address)
```

---

### RET Expansion

```text
address = pop()
jump(address)
```

---

# 3. Active Recall Questions

## Fill in the Blank

**Q1.** CPUs fundamentally execute instructions __________.  
**A.** sequentially.

---

**Q2.** The compiler emits actual __________ instructions.  
**A.** machine.

---

**Q3.** Function inlining copies the function __________ into the caller.  
**A.** body.

---

**Q4.** Function outlining stores function code only __________.  
**A.** once.

---

**Q5.** The essential instruction needed for function calls is __________.  
**A.** jump.

---

**Q6.** A stack uses __________ order.  
**A.** Last-In, First-Out (LIFO).

---

**Q7.** The stack pointer stores the address of the __________.  
**A.** topmost stack element.

---

**Q8.** PUSH typically __________ the stack pointer.  
**A.** decrements.

---

**Q9.** POP typically __________ the stack pointer.  
**A.** increments.

---

**Q10.** CALL = PUSH + __________.  
**A.** jump.

---

## Short Answer

**Q11.** Why does hardcoding a return address fail?  
**A.** A function can be called from multiple locations.

---

**Q12.** Why does a single dynamic return variable fail?  
**A.** Nested calls overwrite it.

---

**Q13.** Why is the stack ideal?  
**A.** Return order naturally matches LIFO.

---

**Q14.** Why must stack logic initially be inlined?  
**A.** Helper routines would require function calls before function calls exist.

---

**Q15.** Why use a dedicated stack pointer register?  
**A.** To avoid memory accesses and improve speed.

---

**Q16.** What is ABI?  
**A.** Application Binary Interface defining calling conventions.

---

**Q17.** Why might large functions not be inlined?  
**A.** Code size explosion.

---

**Q18.** Why can inlining improve cache performance?  
**A.** Caller and callee may stay in instruction cache together.

---

**Q19.** Why can outlining improve cache performance?  
**A.** Rarely used code stays outside hot execution path.

---

**Q20.** Why is recursion impossible to inline generally?  
**A.** Unknown recursion depth at compile time.

---

# 4. Critical Thinking & Application Questions

### Basic

1. Why does LIFO ordering naturally solve function-return problems?
2. Could a queue implement function returns? Why or why not?
3. Why are jumps sufficient for implementing all control flow?

---

### Intermediate

4. Design CALL and RET using only primitive jumps and memory.
5. Compare register-based and memory-based argument passing.
6. How does ABI compatibility allow separately compiled code to work together?
7. What tradeoffs exist between executable size and runtime speed?

---

### Advanced

8. How would cache-line size influence inlining heuristics?
9. How could branch prediction interact with outlined functions?
10. Prove that recursion requires per-call storage isolation.
11. How could tail-call optimization eliminate some stack usage?
12. Why might inlining worsen performance despite removing jumps?
13. Design an alternative stack-growing-upward architecture.
14. How would multi-threading affect stack design?
15. How do local variables relate to stack frames?

---

# 5. Common Pitfalls, Edge Cases & Misconceptions

- Misconception: CPUs inherently understand “functions.”
- Reality: functions are compiler-generated control-flow patterns.

---

- Misconception: CALL and RET are fundamental.
- Reality: they are convenience instructions built atop stack operations.

---

- Pitfall: Hardcoded return address.
- Breaks with multiple callers.

---

- Pitfall: Single return-address variable.
- Breaks with nested calls.

---

- Pitfall: Assuming inlining is always faster.
- Can cause code bloat and cache degradation.

---

- Pitfall: Assuming outlining is always smaller.
- Tiny frequently called functions may incur more overhead.

---

- Pitfall: Passing all arguments through registers.
- Impossible when arguments exceed register count.

---

- Pitfall: Recursive parameter storage in fixed memory.
- Causes overwrite across recursive levels.

---

# 6. Concept Connections & Mind Map

Function execution is built from simple jumps plus state preservation. The compiler injects mechanisms for storing return addresses, passing parameters, and restoring control flow. The stack emerges naturally as the correct abstraction for nested calls and recursion. Hardware later optimizes this abstraction with stack-pointer registers and dedicated CALL/RET instructions. Compiler optimization then balances inlining and outlining based on performance, code size, and cache behavior.

```text
Function Execution
│
├── Compiler
│   ├── emits machine code
│   ├── inserts stack logic
│   └── chooses inline vs outline
│
├── Control Flow
│   └── Jump Instructions
│
├── Function Outlining
│   ├── needs return address
│   ├── needs stack
│   ├── CALL
│   └── RET
│
├── Stack
│   ├── push
│   ├── pop
│   ├── stack pointer
│   └── recursion support
│
├── Parameter Passing
│   ├── registers
│   └── memory
│
└── Optimization
    ├── inlining
    ├── code size
    └── cache behavior
```

---

# 7. Quick Reference Cheat Sheet

### CALL Mechanics

```text
push(return_address)
jump(function)
```

---

### RET Mechanics

```text
address = pop()
jump(address)
```

---

### Stack Operations

#### Push

\[
SP = SP - 1
\]

```text
MEM[SP] = value
```

---

#### Pop

```text
value = MEM[SP]
```

\[
SP = SP + 1
\]

---

### Inlining

✅ No jumps  
✅ Faster  
✅ Better locality  

❌ Larger binary  
❌ Impossible for recursion  

---

### Outlining

✅ Smaller binary  
✅ Supports recursion  
✅ Better for large/rare functions  

❌ Jump overhead  
❌ Stack overhead  
❌ Possible cache misses  

---

### Argument Passing

**Registers**
- Fast
- Limited capacity

**Memory**
- Flexible
- Slower

---

### Key Principle

> CALL and RET are abstractions built on top of **stack + jumps**.

---

**End of Notes**
