---
layout: instance/blog-post
title: "Solr integration tests with Docker and Testcontainers"
tagline: "super duty, super safe"
featured: false
image: /assets/images/blog/zion.jpg
---

When developing Solr plugins for the open source community
(or your in-house Solr team) you're faced with testing the
plugin's compatiblity against a range of Solr versions.
The [Solr Test Framework](https://mvnrepository.com/artifact/org.apache.solr/solr-test-framework) provides a great tool set for
writing JUnit and integration tests against a _single Solr
version_.

In this post I'll guide you through adding Integration
Tests to a Solr plugin project. We want to ensure the
plugin's functionality through a configurable range of
Solr versions. I will add links to working examples in
the [Querqy Query Parser](https://github.com/querqy/querqy) as I recently added those
kind of tests there.

<!--more-->

## 1. Integration testing

When it comes to integration testing, we strive to test our
build artifact in possible runtime environments. In our case
these are different versions of Solr.

> But: **D**on't **R**epeat **Y**ourself: Don't test functionality
that you already tested in your unit tests again! Use a simple
test setup to verify integration capabilities.

In our integration tests, we'll spin up a Solr server and load a
predefined product catalog into a collection. Then we'll check if
the plugin handles the configured query rewrites correctly.
Tools used are:

* [Maven](https://maven.apache.org/) as build tool,
* [Docker](https://www.docker.com/get-started) to run Solr containers in and
* [Testcontainers](https://www.testcontainers.org/) as extension to [Junit](https://junit.org/junit4/) to spin up Docker containers

### 1.1 Marking integration tests

When building and packaging our plugin, we wan't to execute the
integration tests after running the unit tests and after packaging
the plugin. To achieve that, we need to distinguish Unit from
integration tests. We'll use [JUnit categories](https://github.com/junit-team/junit4/wiki/Categories) for that.

We create a empty marker interface `cool.solr.plugin.IntegrationTest`
and mark the integration tests as `@Category(IntegrationTest.class)`

### 1.2 Running integration tests in Maven

In Maven we configure the _maven-surefire_plugin_ (Unit Tests) and
_maven-failsafe-plugin_ (Integration Tests) in the `pom.xml` to

* exclude our marked test classes during unit testing and
* include  them during integration testing:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.0.0</version>
    <configuration>
        <!-- exclude integration tests -->
        <excludedGroups>cool.solr.plugin.IntegrationTest</excludedGroups>
    </configuration>
</plugin>
<plugin>
    <artifactId>maven-failsafe-plugin</artifactId>
    <version>3.0.0</version>
    <configuration>
        <includes>
            <include>**/*</include>
        </includes>
        <!-- run integration tests only -->
        <groups>cool.solr.plugin.IntegrationTest</groups>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>integration-test</goal>
                <goal>verify</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

## 2. Testcontainers

The [Testcontainers project](https://www.testcontainers.org/) is the swiss army knife for launching
Docker containers from JUnit tests. You need to add a dependency
to Testcontainers in your _Maven POM_. It makes sense to add the
[Testcontainers Solr module](https://www.testcontainers.org/modules/solr/) as well.

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <version>1.18.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>solr</artifactId>
    <version>1.18.0</version>
    <scope>test</scope>
</dependency>
```

Testcointainers provides a convenient `GenericContainer` wrapper
for loading, launching and destroying Docker containers from within
JUnit. To launch a Docker container from your JUnit test, define add a
`GenericContainer` instance variable annotated as `@Rule` or
`@ClassRule` to your test.
[Junit rules](https://www.baeldung.com/junit-4-rules) are instanciated
before your Junit test `@Before` methods are called. Most important
they are shut down properly after the test finished.

```java
@Category(IntegrationTest.class)
public class SolrQuerqyIntegrationTest {

    @ClassRule
    public SolrContainer solr = new SolrContainer(DockerImageName.parse("solr:8.11.2"));

    [...]
}
```

The code snippet above launches a _Solr 8.11.2_ Docker container before
our tests starts, executes all test method in the test class and shuts
it down afterwards.

## 2.1 Extending Testcontainers

Now we're going to test our freshly built plugin in a off-the-shelf
Solr container. We could build a custom Solr container (tedious and time consuming) or bind mount artifacts into the Docker container
during startup.

Testcontainer's `GenericContainer` has hooks to do so and the
`SolrContainer` already makes use of them. We created a
`QuerqySolrContainer` that inherits from `SolrContainer` (and
transitive from `GenericContainer`). During construction, we
need to mount two items into the starting Docker container:

* the _recently built and packaged plugin jar_ artifact
* the prepared _test configset_


```java
public class QuerqySolrContainer extends SolrContainer {

    private static final String PROP_PROJECT_BUILD_FINALNAME =
        "project.build.finalName";
    private static final String PROP_PROJECT_BUILD_DIRECTORY =
        "project.build.directory";

    public QuerqySolrContainer() {
        super(DockerImageName.parse("solr:8.11.2"));

        String querqyBinaryPath = String.format("%s/%s-jar-with-dependencies.jar",
                System.getProperty(PROP_PROJECT_BUILD_DIRECTORY),
                System.getProperty(PROP_PROJECT_BUILD_FINALNAME));
        String querqyConfigurationPath = String.format("%s/test-classes/integration-test/%s",
                System.getProperty(PROP_PROJECT_BUILD_DIRECTORY),
                "chorus");

        // link querqy binary into container
        addFileSystemBind(querqyBinaryPath,
            "/opt/solr/server/solr-webapp/webapp/WEB-INF/lib/querqy.jar",
            BindMode.READ_ONLY);

        // link conf directory into container
        addFileSystemBind(querqyConfigurationPath,
            String.format("/opt/solr/server/solr/configsets/%s", QUERQY_IT_CONFIGSET),
            BindMode.READ_ONLY);
    }
}
```

> See [QuerqySolrContainer](https://github.com/querqy/querqy/blob/master/querqy-for-lucene/querqy-solr/src/test/java/querqy/solr/it/QuerqySolrContainer.java#L41-L59) in the Querqy project for a full example.

As plugin versions will change over time, we inject the
[Maven build properties](https://books.sonatype.com/mvnref-book/reference/resource-filtering-sect-properties.html) into the
Junit test to locate the currently built JAR. Unfortunately
these properties are not accessible by default, so we need to
explicitly transfer them as `systemPropertyVariables` in our
Maven POM:

```xml
<plugin>
    <artifactId>maven-failsafe-plugin</artifactId>
    <version>3.0.0</version>
    <configuration>
        <includes>
            <include>**/*</include>
        </includes>
        <groups>cool.solr.plugin.IntegrationTest</groups>
        <systemPropertyVariables>
            <project.build.directory>${project.build.directory}</project.build.directory>
            <project.build.finalName>${project.build.finalName}</project.build.finalName>
            <project.version>${project.version}</project.version>
       </systemPropertyVariables>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>integration-test</goal>
                <goal>verify</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

Building the project now launches a Solr 8.11 Docker container with
our project artifact and test configset mounted. We can now overwrite
the `containerIsStarted` hook in `GenericContainer` to e.g. upload
the mounted configset. If needed, this is the place to create a
collection or import test product data.

```java
protected void containerIsStarted(InspectContainerResponse containerInfo) {
    try {
        // upload configset that we linked into the container
        ExecResult result = execInContainer("solr", "zk", "upconfig",
            "-z", "localhost:9983",
            "-n", "chorus",
            "-d", String.format("/opt/solr/server/solr/configsets/%s", "chorus"));
            if (result.getExitCode() != 0) {
                throw new IllegalStateException(
                    String.format("Could not upload %s configset to Zookeeper: %s", "chorus", result));
            }
    }
}
```

### 2.2 Testing multiple Solr versions

We now have a fully running integration test for our Solr plugin.
In this step we want to run the same test case for multiple Solr
versions. We use the [Junit Parameterized](https://github.com/junit-team/junit4/wiki/Parameterized-tests) extension for the job:

* The _Solr versions to test against_ are injected via the system
  property `solr.test.versions` as a comma separated list
* Use _Parameterized Junit runner_ to run same test for different Solr
  versions

> Find the details how to use the Parameterized Junit runner in
> the commented code snippet below

```java
// Enable the Paramterized runner
@RunWith(Parameterized.class)
@Category(IntegrationTest.class)
public class SolrQuerqyIntegrationTest {

    // Supply a stream of parsed DockerImageName that
    // we extract from a system property
    @Parameters(name = "{0}")
    public static Iterable<? extends Object> solrVersionsToTest() {
        return Arrays.asList(
            System.getProperty("solr.test.versions")
                .split(","))
                .stream()
                .map(image -> DockerImageName.parse(image))
                .collect(Collectors.toList());
    }

    // this gets constructed per parameterized test run
    @Rule
    public QuerqySolrContainer solr;

    // for each run of the test, the parsed DockerImageName
    // is supplied as constructor argument to the test case
    public SolrQuerqyIntegrationTest(DockerImageName solrImage) {

        // we delegate the given DockerImageName to our custom
        // Solr container implementation
        this.solr = new QuerqySolrContainer(solrImage);
    }

    [...]
}
```

In our Maven POM we can now add a `systemPropertyVariables` entry named
`solr.test.versions` with a comma separated list of Solr versions
to test with:

```xml
<plugin>
    <artifactId>maven-failsafe-plugin</artifactId>
    <version>3.0.0</version>
    <configuration>
        <systemPropertyVariables>
            [...]
            <solr.test.versions>solr:8.11,solr:8.10,solr:8.9,solr:8.8,solr:8.7,solr:8.6,solr:8.5,solr:8.4,solr:8.3,solr:8.2,solr:8.1</solr.test.versions>
        </systemPropertyVariables>
    </configuration>
    [...]
</plugin>
```

## Summary

We have to combine and stir a whole lot of applications, libraries and
technologies, but it's worth it. In terms of Querqy, releases are now
more mature and confident as we detect Solr version compatiblity
problems at an early stage.

The integrations tests are run using [Github Actions](https://github.com/querqy/querqy/blob/master/.github/workflows/integration-tests.yml)
and verify every commit an pull request.
