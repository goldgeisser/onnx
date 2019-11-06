#   Support dynamic shape operators and tensors

## Introduction 

Handling Dynamic shapes in accelerated devices is a challenging task – different vendors may implement different solutions.  The goal of this discussion is to propose a uniform handling of Dynamic Shapes among all vendors. 

We propose to implement a flag/option that would enable the vendors replaces a dynamically shaped output with a statically defined shape. The issue involves three different cases:

#### •	inferable,
#### •	non-inferable
#### •	dynamic inputs (to the model).

Inferable operators like shape, slice and resize are arguably the easiest to handle. An acceptable example of dealing with them is the approach by Tensor RT version 6.

The other two test cases are more challenging. Non-inferable operator’s output depends on the content of the input as in the case of NonZeros, Unique and NMS operations and dynamic input affect the entire intermediate model’s tensors. In this proposal we would like to discuss all three cases.

## Implementation Considerations and Discussion:
### Dynamic shaped operators:

1.	Non – Inferable:
a.	NonMaxSuppression
b.	NonZero
c.	Unique

2.	Inferable:
a.	Resize
b.	Shape
c.	Slice
d.	Gather
e.	
### How we define what is dynamic
Handling dynamic shapes can be done at global scope or per operator.
•	The former requires to add a flag field to the tensor proto which indicates if a tensor is dynamic or static. If a tensor is dynamic, the “dims” information is considered a directive hint from the model creator regarding the maximal value on each dimension, yet the actual may be lower. It is up to the implementation to provide shape propagation mechanism to pass the actual dimensions between operator’s outputs and inputs.
o	Issues: what happens if the user defined a dynamic shaped tensor that enters a dynamic operator (e.g., NMS) can we solve this without informing the operator somehow?
•	The latter require adding a flag to each operator. Similar to TensorRT implementation this might results in more than one flag.
•	
### Dynamic inputs
In many topologies the input shape may be variable thus changing the entire model’s intermediate tensors shapes.
Questions:
•	Should we have a different implementation to the input Tensor? As it’s affects the entire model. How do we do it with a flag field to the tensor proto.
Tensor RT solution is to ask the user to specify different Optimization profiles: each profile describes a range of dimensions for each network input, and the dimensions that the graph compiler should use for optimization. This sort of implementation doesn’t require any change from ONNX .
### Who specify the max and what happens in case of violation?

Assuming a vendor decides to set a max value lower than the worst-case scenario value based on some data statistics (e.g., nonzero operator with max size lower than the size of the tensor). What happens if the max value is violated?
Suggestions: Passing only the first (implementation defined) max values. Thus, in those cases different vendors might get different results on the same model

### Different dimensions can change what do we want to do about it?
We might have to support indication for each dynamic dimension of a tensor. Should we insist of any restrictions?

### Who is responsible for shape propagation?
1.	ONNX responsible to define a protocol per dynamic operator.
2.	Custom op – the extension interface should support it.

## Test Cases:
1.	Inferable - Shape
2.	Non-Inferable – content dependent
a.	NMS
b.	NonZero

## Full proposals
Please fill full proposal
## Qualcomm comments – should be removed

Providing feedback on the proposal from Qualcomm's perspective:
In general, we agree that there is a need for better handling of dynamic shapes, especially when it comes to HW accelerators or highly efficient inference runtimes. However, we have a slightly different view on it, and would like to evaluate what needs to be added/modified at the ONNX container level, vs what is part of implementation-specific details.
We support the suggestion for defining “max_output_boxes_per_class” as const value which does not change between inferences, and used as a hint to accelerators on the maximal size to allocate.
•	Following the recent guidance from the operators SIG, we would recommend that this input would become an attribute on next operator version (fixed/const values should be attribute, not inputs)
•	Alternatively, on the next version this argument can be removed completely if our suggestion below for tensor modification would be accepted.
We see the suggestion to add sentinel values (e.g. padding) to indicate non-used values as less appealing, and suggest not to adopt it.
•	Qualcomm Neural Processing SDK and Hexagon NN SDK used to have sentinel values on older implementations/operations, and moved to a newer method of indicating the actual/valid shape as part of the input/output tensors.
•	Specifying the actual shape per inference as part of the tensor information is more robust and does not require “per-operation” special awareness of the sentinel values. It also may be more performant on some implementations.
Regarding the proposal to add a flag to indicate dynamic shape, we would like to suggest an alternation of this idea. We believe that handling dynamic shapes should be done at a global scope, not per operator. As such, our suggestion would be to add a flag field to the tensor proto (https://hexdocs.pm/onnxs/Onnx.TensorProto.html) which indicates if a tensor is dynamic or static.
•	If a tensor is dynamic, the “dims” information is considered a directive hint from the model creator regarding the maximal value on each dimension, yet the actual may be lower.
•	It is up to the implementation to provide shape propagation mechanism to pass the actual dimensions between operators outputs and inputs.

