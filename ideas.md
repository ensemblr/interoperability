# Ideas for Language Interoperability Framework Development


### Notes

The idea is to create a framework that is able to take-in code of any language that it supports. The code input into the framework should be able to take any language's code as input. (similar to the .NET architecture)

There is also a more specific requirement of creating an ensembling framework. This framework should be able to take in a few different kinds of classifiers that can be in any languages supported by the framework. This framework should be as inclusive as possible, so that various kinds of tools can be utilized and less rigid structure can be used.

Possible Approaches :

1. CORBA, SOAP (probably the most comprehensive (and most complex) approach)
2. Web services and other service-oriented architectures (probably the most modern approach)
3. IPC; Inter Process Communication

An important part of the framework will be a parallel, cluster based system that can combine and manage multiple machines for the working of code. This can be done using MPI (Message Passing Interface).

Also, DSL (Domain Specific Language) based graph parsing, graph optimization and graph clustering algorithms (DAGs) can be used to make these things faster.


## Sharing Operators between DL Frameworks

This discussion started from [dmlc/minpy#129](https://github.com/dmlc/minpy/issues/129), with [@soumith](https://github.com/soumith) THC is a tensor library that backs torch. I open this issue in MXNet repo so more developers can see it.

First of all, it is possible reuse operator libraries between frameworks, for example

Support for THC and Torch Module was done in [Torch Plugin](https://github.com/dmlc/mxnet/tree/master/plugin/torch), with interfacing to torch's lua library.
MXNet supports reuse operators from [caffe](https://github.com/caffe/caffe)
It is always interesting to see interchangeability happen. For example, schedule pytorch operations in mxnet's async engine, or run mxnet's declarative API to directly share data with pytorch's array.

However, there is some engineering obstacles in doing so, which I would like to explain what these obstacles are, and hopefully this can motivate the community to move forward, and make this easier.

### Coupled Operator Data Structure Components

An operator can mean many things, here are some basic components on what the operators are:

Data structure that holds(shape) pointers to the array
Possible memory allocator to handle run-time memory allocation
Resource handles, if external resources is needed
Scheduling related objects if array support synchronize execution
Why such coupling prevents reuse? There are two reasons

Many systems have their own memory allocator and ways of resource handling code.
While having memory allocator enables runtime memory allocations, sometimes memory allocation is not preferred at all(e.g. BLAS calls where all memory are pre-allocated)
To resolve this problem, an operator library design should enable operators that accept user managed memory resources, when possible, not introduce allocator or resource management, but give hints to the user(CuDNN's workspace requirement eliminates the need to internal memory allocator).

From this point of view, CuDNN an cuBLAS are good examples. THC is nice, but still encapsulate memory allocator(which is needed sometimes for dynamic operators).

### Lack of Unified Operator Interface

The second obstacle is mainly lack of common operator interface. This is a problem of CUDNN and THC that prevents reusing. Take CuDNN for example, each CuDNN API is a C function, with its own interface, to adopt the operator, there need to be one(or multiple) adapting function per operator.

Consider instead, if there is an unified operator interface(the following is a mock design), where each TBlob is a reference to the data fields and shape, and every function gets registered to the registry with their name

using FCompute = std::function<void (
   array_view<TBlob> ins, array_view<TBlob> outs, map kwargs, stream stream)>
Then it only takes one function to extract, and reuse all operators and automatically expose them to front end. In MXNet, it even directly generates the symbolic counterpart from the same imperative operator, if gradient is provided.

### Problem of One Unified Operator Interface

There is always a flip side of the coin. Assume that we go with a unified operator interface. As a matter of fact, that is what MXNet, TensorFlow and Caffe have done. The problem now becomes what the interface should look like? One trap that framework designer always falls into is that we need one interface that rules them all.

Since one interface rules them all, we want to support all possible operators, what about the ones that need runtime memory allocations? Maybe add memory allocator to it, what about the ones that is asynchronize? In the end, the interface have to include memory-allocator, scheduling module in some way,
and that introduces the "Coupled Operator Data Structure Components" problem. The operator interface become deeply coupled with the rest of the framework and not reusable.

### A Better Solution: A Few Unified Interfaces

Can we get the best of both worlds, having as few data structures and interfaces as possible, while still not introducing coupling to allocator and scheduling as much as possible? I think the answer is yes and we need to jump out from the ideal of one interface that rules all the operators.

I can categorize the operators roughly in three categories

type1: Basic operators: The ones that can do shape inference based on input shape, can take memory pointer, stream and go
type2: Basic+ operators: Same as basic operator, but also need to declare some additional resources(workspace)
type3: Complicated operators: The ones that requires runtime memory allocator, its output shape depends on content of the data.
If we design for general operator interface, the answer will usually looks like type3. However, type 1 and 2 dominates 90%+ of the major operators we are using.
If we design one operator interfaces for each type, this problem is solved. So that frameworks can pull and interact with each type in their own way.
It is much easier to do things like static memory planning if type1 and type2 are explicitly introduced. This is one additional layer of wrapping on top of THC and CuDNN is is lacking so far.

A registry system like NNVM could come very handy to easily resgister these informations, and get pull out by the libraries.

### The Hope

I have always hopped that there is a minimum set of operator interface standard in C++, that can be shared across libraries. I think we have a good idea on what the solution looks like. While most system tends to become opague and coupled, I think this kind of transparent way can help evolve the community in a healthy way. This being said, there is always effort to make these happen. This involves a open discussion on what the interfaces should be and commitment from framework builders. I would really love to see this happen, and that is why I spend more than one hour writing this.

Unfortunately, most frameworks already have kinda of "enough collection of operators", so having a unified operator interface will contribute little to each framework in terms of usability in short term. Naturally this would be given lower priority. That is why commitment is needed to bring this out for longer term benefit
