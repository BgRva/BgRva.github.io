---
layout: post
title: "Snowball Extraction of a Sub-Network"
date: 2015-02-19 09:32:53 -0400
comments: true
categories: [graphs]
---

What if you need a sub-network from a larger one, and how would you extract it?  I'll walk you through the steps of how to do a Snowball extraction, (or extraction by steps).  I have no doubt this goes by many names, but Snowball is the name I've been familiar with for years.

Assume you are given a network, and by network I mean a graph structure with data attributes on the nodes and/or edges such that the graph represents data connected with respect to a particular context.  For example, take a simple graph:

{% img /images/simple-graph.jpg 326 254 'simple graph' %}

Add some names and now it is a simple social network:

{% img /images/simple-network.jpg 326 254 'simple social network' %}

Graph + Data = Network

###A Snowball Extration

Lets start with the following example network and perform a snowball extraction 3 steps/hops out.

{% img /images/network-0.jpg 629 418 %}

Step 1: given a set of seed nodes: (2, 16), find all nodes 1 step or hop out from the seed nodes.  The nodes we are interested in are the neighbors of the seed nodes along *outgoing* edges, (in this example we will be considering the directionality of the edges).  Seed nodes are in green:

{% img /images/network-1.jpg 629 418 %}

Step 2: Now gather the new nodes found in Step 1: (1, 3, 9, 10, 36) and repeat the process: 

{% img /images/network-2.jpg 629 418 %}

Step 3: Get the new nodes from step 2: (4, 6, 8, 1, 12, 17, 34) and repeat: 

{% img /images/network-3.jpg 629 418 %}

Step 4: With the new nodes from step 3: (5, 7, 13, 15, 18, 19, 33), combine all found nodes and edges for the resulting sub-network:

{% img /images/network-4-final.jpg 629 418 %}

Note that using different seed nodes you may end up with a drastically different subnetwork and much depends on the structure and properties of the main network.  Additionally, how you handle duplicate edges, self-loops, and parallel-edges are completely determined by your problem at hand.  I've attempted to formalize the steps as pseudo-code below, but again the complexities of the problem at hand will affect how this should be implemented.

{% codeblock lang:C# pseudo-code%}

foundNodes {}   // all nodes found in the snowball so far
currentNodes {} // the current nodes from which we are stepping out
currentEdges {} // the edges we find outgoing from the current nodes
maxStep = 3
currentStep = 1

nextNodes = {SEEDS}  // this should be initialized to the seeds 4,10

while (currentStep <= maxStep)
  currentNodes = nextNodes;
  foundNodes.Add(currentNodes);
  
  foreach each edge
    If edge.SourceId  is in currentNodes
      currentEdges.Add(edge)

  nextNodes = {}
  foreach edge in currentEdges
    If edge.TargetId is NOT in foundNodes
      nextNodes.Add(edge.TargetId)

{% endcodeblock %}



