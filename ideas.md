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

