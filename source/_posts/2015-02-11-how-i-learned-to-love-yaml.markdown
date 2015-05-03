---
layout: post
title: "How I Learned to Love YAML"
date: 2015-02-11 11:01:01 -0400
comments: true
categories: [YAML, C#]
---

This post describes how I came to use [YAML](http://www.yaml.org/) to handle complex data scenarios for integration test cases.

As a proponent of TDD, generating test data is a necessity.  When the units under test increase in complexity, so does the test data.  Test data can range from simple input strings used in [NUnit TestCaseAttribue](http://www.nunit.org/index.php?p=testCase&r=2.5) entries to custom test scenario objects which included raw data as well as expectations of varying outcomes.  

Generating complex test scenarios was especially the case when I was working on an application that did graph manipulation and required converting raw data to graph structures.  Because of the variations in graph structures that had to be handled when converting from input data, the test scenarios had to provide the raw data to be converted to a graph, the expected graph structure, and other expectations.  These custom scenario objects required creating code to read them in, to access the data and expectations, as well as managing changes  without requiring brain surgery on the codebase. 

As an example of the range of test data needed for converting data to graphs,  here are 4 trivial examples of input data and the expected graph:

{% img /images/graph-variations.jpg 'graph examples' %}

The combinations of different sub-cases resulted in a multiple scenarios that needed to be built and managed. To compound things, each test case's data was hand jammed based on a known input an expected output. 

In the past, I would end up with some fairly easy to read text file format and multiple parsers to ingest the files and create the scenario objects to feed the tests.  While the parsers themselves were simple and required limited error handling, the requirements of the scenarios would inevitably change over the course of the project.  For example, let’s say based on some new functionality, in order to implement the tests we needed to know the expected number of *self-loops* (an edge that has the same source and target node) for an input graph.  This would require adding a new property to the scenario object, updating the parsers, and verifying the parsers worked.  While not difficult or risky, these updates still sponge up valuable time and resources.

How come I didn’t format the data as XML or JSON?  Because the integration tests requiring the complex test data grew out of need rather than full up-front awareness and planning (these being developer level integration tests arising from TDD).  The data usually started out for use in simple unit tests, and over time it was aggregated to form a scenario.  The code to support the scenarios grew and morphed as the tests and the project code did.

Here is where [YAML](http://www.yaml.org/) enters the picture.  Very briefly, YAML (*YAML Ain't Markup Language*) is a human friendly parse-able specification for which there exist parsers in multiple languages.  In other words, perfect for these test data scenarios.  There are many, many, awesome tutorials on it, so I will simply focus on how I used YAML to handle the scenario data.  For the examples below, I am using [YamlDotNet](https://github.com/aaubry/YamlDotNet) by Antoine Aubry.  Though YAML.org identifies other .Net parsers, they have not been updated in quite some time. On top of that, YamlDotNet is very nice.  


Here is the specification of a test scenario in YAML for example #4:

{% img /images/graph-variation-4.jpg 'graph example 4' %}

{% codeblock  lang:yaml scenario 4 as yaml %}
---    
    id: 4
    node-content-format: Name
    edge-content-format: Empty
    description: scenario to test parallel edges and self-loops including an orphan
    raw-nodes: 3
    raw-edges: 4
    expected-nodes: 3
    expected-edges: 4
    expected-self-loops: 2

    nodes:
        - id:    0
          content: (LEELA)
          
        - id:    1
          content: (FRY)

        - id:    2
          content: (BENDER)
                    
    edges:
        - id:         0
          source-id:  0
          target-id:  1
          content:    ()
          
        - id:         1
          source-id:  1
          target-id:  0
          content:    ()
          
        - id:         2
          source-id:  1
          target-id:  1
          content:    ()

        - id:         3
          source-id:  2
          target-id:  2
          content:    ()
                    
    expected-structure: [0,1,0]
                        [1,1,0]
                        [0,0,1]

    expected-edge-structure: []  [1] [0]
                             [1] []  []
                             [1] []  []

{% endcodeblock %}

This test data was parsed in to a Scenario class instance that was used to feed integration tests:

{% codeblock  lang:C# Scenario class structure %}

  public class Scenario
  {
    public Scenario()
    {
      Nodes = new List<NodeIndicator>();
      Edges = new List<EdgeIndicator>();
    }
 
    public int Id { get; set; }
    public string Description { get; set; }
    [YamlAlias("node-content-format")]
    public ScenarioNodeContentFormat NodeContentFormat { get; set; }
    [YamlAlias("edge-content-format")]
    public ScenarioEdgeContentFormat EdgeContentFormat { get; set; }
    public List<NodeIndicator> Nodes { get; set; }
    public List<EdgeIndicator> Edges { get; set; }
    [YamlAlias("raw-nodes")]
    public int NodeCountRaw { get; set; }
    [YamlAlias("raw-edges")]
    public int EdgeCountRaw { get; set; }
    [YamlAlias("expected-nodes")]
    public int NodeCountExpected { get; set; }
    [YamlAlias("expected-edges")]
    public int EdgeCountExpected { get; set; }
    [YamlAlias("expected-self-loops")]
    public int ExpectedSelfLoops { get; set; }
    [YamlAlias("expected-structure")]
    public string ExpectedStructure { get; set; }
    [YamlAlias("expected-edge-structure")]
    public string ExpectedEdgeStructure { get; set; }
  }

  //---------- 
  public class EdgeIndicator
  {
    [YamlAlias("source-id")]
    public int SourceId { get; set; }
    [YamlAlias("target-id")]
    public int TargetId { get; set; }
    public string Content { get; set; }
  }
 
  //---------- 
  public class NodeIndicator
  {
    public NodeIndicator()
    {
      Id = -1;
    }
    public int Id { get; set; }
    public string Content { get; set; }
  }

  //---------- 
  public enum ScenarioNodeContentFormat
  {
    Empty,
    Name,
  };

  //---------- 
  public enum ScenarioEdgeContentFormat
  {
    Empty,
    Name,
  };
 
 
{% endcodeblock %}

Some explanations about the Scenario class:  

* _NodeIndicator_ and _EdgeIndicator_ are objects that 'indicate' how a node or edge are to be constructed or connected, not the nodes or edges themselves.  It is up to the calling code that builds the graph to interpret this.
* The _Content_ property for each indicator represents a string into which different data can be associated with a node or edge.  For example, each node has a content value of a name surrounded by parenthesis:

{% codeblock lang:yaml  %}
        - id:    0
          content: (LEELA)
{% endcodeblock %}

* The format of the node and edge Content field is specified by the _NodeContentFormat_ and _EdgeContentFormat_ properties (respectively)
* The _ExpectedStructure_ is a string representation of the expected structure of the graph once it is constructed.  The string is a row-major representation of the graph as a matrix.
* The _ExpectedEdgeStructure_  is a string representation of the expected connectivity of the edges in the resulting graph.  The format is as follows
  * Each row represents the edges indicent to a node starting at node index 0
  * The contents of each row are
    * First '[]' is an array of edge ids for self loops (* self loops are not included in the inbound or outbound arrays ).
    * Second '[]' is an array of edge ids for inbound edges
    * Third '[]' is an array of edge ids for outbound edges

Now, if I need to add new data fields to support new tests, then only tricky thing is getting the YAML syntax correct in the data files and of course adding any new properties to the Scenario class. The parsing is handled by YamlDotNet no brain surgery is required.  If only I'd found YAML earlier.



