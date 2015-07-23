Workload Generation
===================

Overview
--------

This tool is a workload generator for testing the performance of
mongod. The tool allows you to specify a workload in a yaml file, with
a large degree of fleibility. The workload tool is written in C++ and
wraps the C++11 mongo driver, which supports the mongo driver spec.

There are three basic components in a workload, enabling the
specification of arbitrarily complex workloads in a graph of
operations. 
1. Operations: These are basic operations, such as a find_one. These
   operations *do* something
2. Nodes: These are nodes in the workload graph. They often have an
   associated operation. Nodes control the order of execution of the
   operations.
3. Workloads: These are collection of nodes and represent an actual
   workload. They specify the node that should execute first in the
   workload. They also supply parameters and a pseudo-random number
   generator to the nodes and ops. 

Currently operations are embedded in nodes of the same name. This is
going to be refactored soon that operations are first class objects,
owned by a node. 

### Currently supported operations

1. query
2. insert
3. sleep: sleep for some number of milliseconds. The entry sleep
   specifies how long to sleep. 

### Currently supported nodes

1. basic: This is the default node. It executes its associated
   operation, and then moves execution to its next node.
2. forN: This node wraps a workload. It executes the workload N times
3. randomChoice: This node has a list of other nodes and
   probabilities. It uses the pseudo-random number generator to select
   the next node to execute.
4. Finish: This represents an absorbing state in the graph. The
   workload is finished when the Finish node executes. There is always
   an implicit Finish state included in the workload. 


### Current Non-Obvious limitations

This tool is new, and a lot of things will be going into it, such as
timing measurement. In addition, currently the parser does not handle
converting YAML sequences into bson lists. Don't make a query in a
query node, or a document in an insert node using a sequence. 

Dependencies
------------
* The [C++11 mongo driver](https://github.com/mongodb/mongo-cxx-driver/tree/master)
* [yaml-cpp](https://github.com/jbeder/yaml-cpp)
* A compiler that supports c++11

Building
--------

There is a simple makefile included. Just call make.
    make

The includes from the makefile may need to be updated for your
system. The project will be moving to cmake in the near future, and
that should improve this process. 

Examples
--------

There are a collections of examples in the examples directory. To run
the forN.yml example, simply do:
    ./mwg examples/forN.yml

The workloads are all specified in yaml. An example query node is:

        name : query
        type : query
        query : { x : a }
        next : sleep

The main parts of this are:
* name: This is a label used to refer to the node or operation
* type: This says what the operation should be. Basic operations
  include the operations supported by the C++11 driver. Currently only
  a limited subset is supported.
* query: This is specific to the query operation. The argument will be
  converted into bson, and will be used as the query document passed
  to the C++11 driver when generating the query
* next: This is the node to transition to when completing this
  operation.

A simple workload that does an insert and a query, and randomly
chooses between them:

    seed : 13141516
    name : simple_workload
    nodes :
         - name : insert
           type : insert
           document : {x : a}
           next : choice
         - name : query
           type : query
           query : {x : a}
           next : choice
         - name : choice
           type : random_choice
           next :
               query : 0.5
               insert : 0.5

This workload will run forever, and after each operation make a random
choice of whether to do an insert or query next. The workload can be
made finite by adding an absorbing state.

    seed : 13141516
    name : simple_workload
    nodes :
         - name : insert
           type : insert
           document : {x : a}
           next : choice
         - name : query
           type : query
           query : {x : a}
           next : choice
         - name : choice
           type : random_choice
           next :
               query : 0.45
               insert : 0.45
               Finish : 0.1

The Finish state is an implicit absorbing state. The workload will
stop when it reaches the Finish state. 
