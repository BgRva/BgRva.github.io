---
layout: post
title: "Using NUnit Theory for Integration Tests"
date: 2015-05-14 09:01:11 -0400
comments: true
categories: [graph, C#]
---

NUnit TheoryAttributes ([v2.5](http://www.nunit.org/index.php?p=theory&r=2.5), [v3.0b](https://github.com/nunit/nunit/wiki/Theory-Attribute)) can work well for integration tests and for testing generic methods.  The integration tests I'm referring to are those created by a developer during development:

**Integration Tests** are generated during development, (e.g. TDD), that are applied to units of code that have dependencies on other units</br></br>
**Unit Tests** are small, highly focused tests which are applied to a unit of code in isolation

The precise boundary of when a developer generated test is considered *unit* vs *integration* is squishy at best, and depending who you ask or where you look you will find different answers, ([see M. Fowler](http://martinfowler.com/bliki/UnitTest.html)).  Even with this distinction between the test types, both may be implemented with the same framework (i.e. NUnit).  Ultimately, test early and test often.

Using the TheoryAttribute allows testing a single unit for which the inputs are a bit more complex than simple parameters, and the behavior under test may have a broader range of edge cases.   Integration tests can be used to exercise a method for ‘typical’ scenarios, and I've often found that trying to cover every possible scenario can be unrealistic.  From the NUnit documentation 

> A Theory is a special type of test, used to verify a general statement 
> about the system under development. Normal tests are example-based.        

   (*) as of this post, this statement is applicable to NUnit version 2.5 through the 3.0 beta.  

The code samples below are part of a project to generate Scenario objects that contain the input data for testing the functionality of a graph library (yes, I’m testing a library for testing).  The Scenarios start out as [YAML](http://www.yaml.org/) files and are parsed to an object.  Data (of varying types) can be associated with each scenario and is read in from separate YAML data files.  All of the parsing and hard work is handled by [YamlDotNet](https://github.com/aaubry/YamlDotNet) by Antoine Aubry.  You do not need to worry about YAML or graph functionality in the code samples below, this is just a little background.

**YamlScenarioReader**:  This class reads a scenario data file and returns a scenario object

{% codeblock lang:C# YamlScenarioReader class%}

public class YamlScenarioReader 
{
    public Scenario ReadFile(string fileName)
    {
        Scenario scenario = null;

        var deserializer = new Deserializer(namingConvention: new CamelCaseNamingConvention());

        using (var sr = new StreamReader(fileName))
        {
            scenario = deserializer.Deserialize<Scenario>(sr);
        }

        return scenario;
    }
}

{% endcodeblock %}


Even though YamlDotNet will be doing the heavy lifting, I still want to run some tests for reading in every file as a cursory inspection of the file formats, since the test scenario files tend to change often [post on Yaml](blog/2015/02/11/how-i-learned-to-love-yaml/).  These are an example of what I consider integration tests.

For reading the data, there is a corresponding reader with a T type parameter indicating the type of the underlying data to be read in.  Again, YamlDotNet is doing the hard work, all we need to do is provide a way of validating the files.

{% codeblock lang:C# YamlNodeDataReader class%}

public class YamlNodeDataReader<T>
{
    public Dictionary<int, T> ReadFile(string fileName)
    {
        var data = new Dictionary<int, T>();

        var deserializer = new Deserializer(namingConvention: new CamelCaseNamingConvention());

        using (var sr = new StreamReader(fileName))
        {
            data = deserializer.Deserialize<Dictionary<int, T>>(sr);
        }

        return data;
    }
}

{% endcodeblock %}

The test fixture is as follows:

{% codeblock lang:C# YamlNodeDataReaderFixture class%}

public class YamlNodeDataReaderFixture
{
    [TestFixture(typeof(double))]
    [TestFixture(typeof(DummyItem))]
    public class ReadFile<T>
    {
        [Datapoints]
        public DataFileWrapper<double>[] DataFilesOfDouble = new[]
                    {
                    new DataFileWrapper<double>(@"F:\GraphProject\DataFiles\Data\edge.A.001.double.txt"),
                    new DataFileWrapper<double>(@"F:\GraphProject\DataFiles\Data\edge.A.002.double.txt"),
                    };

        [Datapoints]
        public DataFileWrapper<DummyItem>[] DataFilesOfDummyItem = new[]
                    {
                    new DataFileWrapper<DummyItem>(@"F:\GraphProject\DataFiles\Data\edge.A.001.DummyItem.txt"),
                    };

        [Theory]
        public void Returns_Dictionary(DataFileWrapper<T> dataFile)
        {
            //Arrange
            var reader = dataFile.GetNodeReader();

            //Act    
            var data = reader.ReadFile(dataFile.FileName);

            //Assert
            Assert.NotNull(data);

            var maxKey = (data.Keys.Select(k => k)).Max();

            Assert.AreEqual(data.Count, maxKey + 1);
        }
    }
}

public class DataFileWrapper<T>
{
    public DataFileWrapper() { }

    public DataFileWrapper(string fileName)
    {
        FileName = fileName;
    }

    public INodeDataReader<T> GetNodeReader()
    {
        return new YamlNodeDataReader<T>();
    }

    public IEdgeDataReader<T> GetEdgeReader()
    {
        return new YamlEdgeDataReader<T>();
    }

    public string FileName { get; set; }
}

{% endcodeblock %}

The test method annotated with the *Theory* attribute uses the data properties annotated with *Datapoints* to provide the input data.  Each *Datapoints* property should correspond to one of the types indicated in the *TestFixtureAttributes* atop the *ReadFile<T>* class definition.

**YamlNodeDataReaderFixture** has “fixture” in its name, but it does not have a unit test attribute.  I use the convention of a class for each class under test, then subclasses for each method under test, see [Structuring Unit Tests by Phil Haack](http://haacked.com/archive/2012/01/02/structuring-unit-tests.aspx/). 

**ReadFile<T>** class is the test fixture for testing the ReadFile() method of YamlDotNetReader.  Each TestFixture attribute indicates what the type parameters to use for T.

**DataFilesOfDouble** is a property which returns an array of DataFileWrapper<double> objects for each data file containing data of type double

**DataFilesOfDummyItem** is a property which returns an array of DataFileWrapper<DummyITem> objects for each data file containing data of type double

**Returns_Dictionary()** is the test method to which the input data is applied to the ReadFile() method under test.

**DummyItem** is simply a class used to represent more complex data in the data files.

**DataFileWrapper** is a helper class needed in the example to provide a single generic input parameter to the *Returns_Dictionary()* test method and also provides methods to return which type of reader is under test based on the type of the data.  While not absolutely necessary, it makes the tests cleaner and the maintainability of the tests more robust.

Here is an example of the output (in Resharper) showing how the embedded test classes or each method show up:

{% img /images/rsharper-theory-attrib.png %}

One annoyance I have with parametized Theory tests is that there is no way to distinguish between the individual tests by the respective parameter values.  This can cause a little grief when trying to identify which parameters failed and isolate them, but it is a minor inconvenience for the ability to beat on your code with automated tests. 

