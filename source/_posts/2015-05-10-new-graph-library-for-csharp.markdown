---
layout: post
title: "A New Graph Library for C#"
date: 2015-05-10 10:01:01 -0400
comments: true
categories: [graph, C#]
---

You are probably thinking: *Really*?  Similar libraries exist and they are just fine, so why a new one?  Before I dive in to the pros and cons of this adventure, let me clarify what I mean by "Graph" Library. I'm talking about a [graph](http://en.wikipedia.org/wiki/Graph_theory) as set of nodes and edges where each edge connects two nodes.  There are several libraries available in C# that fall within this scope and they are all quite awesome. A few are listed below, but excluded are graph libraries for visualization or for specific purposes such as ontologies.

  - [Quickgraph](https://quickgraph.codeplex.com/)<br/>
  - [Algorithmia](https://github.com/SolutionsDesign/Algorithmia/)<br/>

The thing about implementing a graph in code is that it is all about compromises. Some implementations may emphasize speed, extensibility, customizability, etc., but each will have its strengths and weaknesses.  For example, here is a simple case of a compromise that may arise when considering a graph implementation as an [adjacency list](http://en.wikipedia.org/wiki/Adjacency_list). Assume we were implementing a graph and wanted to allow consumers of the code to access the nodes by index from 0 to n-1 (where n is the number of nodes in the graph) in O(1) time.  The code samples that follow are simply interface definitions for discussion purposes, implementation details are all hand-waving.

{% codeblock lang:csharp  %}
public interface INode
{
    int Index { get; private set; }
}  

public interface IGraph
{ 
    IList<INode> Nodes { get;}

    int NodeCount { get; }
}  

// Given an instance of IGraph g, access nodes by index
var node1 = g.Nodes[1];
var node2 = g.Nodes[2];

{% endcodeblock %}

How nodes are accessed is coupled with how nodes are added and removed.  Should we allow the user to create a node object and add the node to the graph or should we allow the graph to handle creation of the node?  Each choice has implications for the rest of the implementation.  Consider the following two approaches for adding a node:

{% codeblock lang:csharp  %}
public interface IGraph
{
    List<INode> Nodes { get;}

    int NodeCount { get; }

    /// <summary>
    /// #1 The user creates the node and adds it to the graph object and returns
    /// the graph assigned index of the node.
    /// </summary>
    int AddNode( INode newNode);

    /// <summary>
    /// #2 The graph instance creates and returns the node object, with an assigned index
    /// </summary>
    INode AddNode() 
}

// Given an instance of IGraph g:
// #1
var node1 = new Node();
var index = g.AddNode(node1);
var node = g.Nodes[index];

// #2
var g = new Graph();
var node = g.AddNode();

{% endcodeblock %}

Whereas #1 requires the user to instantiate a node, in #2 the graph maintains more control over instantiation but reduces the flexibility of where and how the user can create a node as the graph object must be instantiated first.  Both approaches still allow for accessing any node by index, but things get sticky when we consider removing a node:

{% codeblock lang:csharp  %}
var g = new Graph();
// create 10 nodes:
for(int i = 0; i< 10; i++)
    g.AddNode();

// 10 nodes - nice; now iterate through them
for(int i = 0; i< g.NodeCount; i++)
    Console.WriteLine(g.Nodes[i].ToString());

//  *********: Here is the issue *********:
g.RemoveNodeAt(4);

// 9 nodes; now iterate through them
for(int i = 0; i< g.NodeCount; i++)
    Console.WriteLine(g[i].ToString())

{% endcodeblock %}

If the *IGraph* implementation provides the ability to access nodes by index from 0 to n-1 and we removed a node, the implementation needs to guarantee the indexing behavior.  This may require re-indexing all the nodes which is an O(n) operation.  If the graph is being implemented specifically for cases where users are just creating graphs from static data and do not require many removal operations, then this could be an acceptable solution.  Realistically, an O(n) operation is not that painful as iterating collections occurs very often.  But for large graphs with significant amounts of Add and Removal operatoins, this could impact user experience.  At the very least, such behavior and its implications should be clearly documented (in sample code as well) for the consumer.

So back to the call for a **New Graph Library for C#**. The driving idea (and need) for implementing this is to provide a general purpose graph structure that is quick to pick up, intuitive to use, and straight forward to represent data.  By "general purpose" we are not aiming to create a graph implementation optimized for narrow use cases (e.g. speed, memory, algorithms, etc.), but rather one that can be used in a wide latitude of cases needing a graph structure (and one which will handle particular issues such as Self-Loops and Parallel Edges).  Additionally, it needs to be small, uncluttered, and it needs to “just work”.  This implies quality documentation and sample code:  Not just more documentation, but the *right* documentation.  Finally, and perhaps most importantly, this will be built for [.Net Core](http://www.dotnetfoundation.org/) so it can run on Mac, Linux, and Windows.

The name will inevitably come out of the initial discussions (though for some reason "dream-graph" with visions of flowers off of the [Mystery Machine](http://scoobydoo.wikia.com/wiki/Mystery_Machine) have been jokingly put forth so far).  More to come and I invite you all to beat on it once the code starts flowing.


