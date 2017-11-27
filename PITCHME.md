@title[Introduction]

# ATE Component Tests

---
@title[Plan]
## Plan

* What is ATE
* Component in testing pyramid
* Evaluation of component test in ATE
* Best practices
---
#### Where will you place component test?
![test pyramid](p1_2.png)

---
#### Component test
![test pyramid](p2_2.png)
---
## Requirements for component tests
* Close to production
* Blackbox 
* Fast feedback
* Stable
* Running locally
* Easy to read, ideally describes what service is doing

Note: 
I worked in a project which did computation on hadoop using hive. There project calculated some advertising metrics, so the numbers themselfs didn't mean a lot to programmers. In order to do a simplest change we needed to run a component test. The test on hadoop took ~1h. It was flaky. The output was just a bunch of numbers, so I coulnd say are they correct or not. morrover there was an invisible simbol separator that made the results completly unreadable. It took about two days to submit a change of 10 minutes coding. It was such a waste of time changing something in that project.
Why am I telling this story - don't do like this.
Since that time I began particulary sensity
---

@title[What is ATE]

## ATE (Simplified)

![small diagram](s_diagram_2.png)

---

@title[What is ATE]

## ATE (zoom in)

![diagram](diagram.png)
Note: diagram here pulse, um, users -> ATE -> users/segments
to undertand the scope of the problem.
---

@title[What is ATE]

### 15 backend engeneers
### 63 gihub repos
### 25 microservices
#### Scala Ec2 S3 Kineis SQS Postgres Aerospike
#### Cassandra Spark ElasticSearch 

---

# 1.5 years ago
### UnitTest 
### IntegrationTest 
### SmokeTest
* Integration test using developers account in  AWS
* Run manually smoke test
* Look at datadog metrics in dev and pre environments

---
## Issues
* Exeptions on startup (not the whole code tested)
* Shared AWS resources between tests
* Issues with local run and security 
* Not very stable
* Slow feedback loop
* Not ready for Continious Deployment

---
## Solution
* Ligtweigt test that test the whole application
* Use dockerised AWS
---

### Shared docker lib

[shared-docker-test-containers](https://github.schibsted.io/spt-advertising/shared-docker-test-containers)
<br />
<br />
based on `com.spotify:docker-client`

---
## Example of a microservice

![calculator](calc.png)
---
## Setup Kinesis
```
class AteOfflineCalculatorComponentTest
  extends FeatureSpec
  with Matchers
  with BeforeAndAfterEach
  with BeforeAndAfterAll
  with DockerForAllTestOps
  with Eventually {

  private val server = new EmbeddedHttpServer(twitterServer = new AteOfflineCalculatorServer)

  private var wireMockServer: WireMockServer = _

  private var userId: String = _

  private val kinesisInputStreamName = "input-stream"
  private val kinesisOutputStreamName = "output-stream"

  private val kinesisContainer =
    registerContainerRunner(
      KinesisContainerDetailsBuilder.withDynamicPort()
        .get.build()
    )
  private val dynamoContainerHolder =
    registerContainerRunner(
      DynamoDbContainerRunnerBuilder.withDynamicPort().build()
    )

  private val aerospikeContainerHolder =
    registerContainerRunner(
      AerospikeContainerRunnerBuilder.withDynamicPort().build()
    )

  override protected def beforeAll(): Unit = {
    super.beforeAll()

    setupWireMock()
    setupKinesis()
    setupAerospike()

    server.start()
    waitForKinesisToInit(server.injector.instance[DatadogClient].asInstanceOf[FakeMetricClient])
  }

  private def setupKinesis(): Unit = {
    val kinesisClient = kinesisContainer.container.newClient()
    createStream(kinesisClient, InputStreamName)
    createStream(kinesisClient, OutputStreamName)
    sys.props ++= Seq(
      "kinesis.consumer.streamName" -> InputStreamName,
      "kinesis.publisher.streamName" -> OutputStreamName,
      "kinesis.endpoint" -> kinesisContainer.container.endpoint,
      "dynamodb.endpoint" -> dynamoContainer.container.endpoint)
  }

  private def createStream(kinesisClient: AmazonKinesis, kinesisStreamName: String): Unit = {
    kinesisClient.createStream(kinesisStreamName, 1)
    (1 to 120).toStream
      .map(_ => kinesisClient.describeStream(kinesisStreamName).getStreamDescription.getStreamStatus)
      .takeWhile(_ != "ACTIVE").last
  }

```
@[6](mix in shared test dependencies)
@[18-20] (create containters)
@[33,37,42] (intialise kinisis)
@[43-52] (update config parameters)
---
## Setup segment definition stub
```
private def setupWireMock(): Unit = {
  wireMockServer = new WireMockServer(WireMockConfiguration.wireMockConfig().dynamicPort())
  wireMockServer.start()
  stubSegmentDefinitionWithSegmentPages()

  val host = s"localhost:${wireMockServer.port()}"
  sys.props += ("spt.segmentDefinition.hosts" -> host)
}

private def stubSegmentDefinitionWithSegmentPages(): Unit = {
  wireMockServer.stubFor(
    get(urlPathEqualTo("/v6/segments"))
      .withQueryParam("page", new EqualToPattern("1"))
      .withQueryParam("page_size", new RegexPattern("[0-9]+"))
      .willReturn(
        aResponse()
          .withStatus(200)
          .withBody(segmentsJson)
      )
  )
}

private val segmentsJson: String =
  s"""[
     |  {
     |    "id": "segment-age-one-bracket",
     |    "value": { 
     |       "ageCriteria": { 
     |         "ageBrackets": [ { "lower": 18, "upper": 25 } ], 
     |         "version": 10
     |       } 
     |    }
     |  },
```
@[1-2,4]
@[10,12,18,21]
@[26-32]

---
## Tests 
```
feature("age criterion matching") {
  scenario("user matches criterion with single age bracket") {
    sendMessageAndVerifyOutput(
      input = UserPropertyDTO.PropertyType.Age(AgeDTO(22)),
      expectedOutput = CriteriaByType(
        CriterionType.AGE, 
        Seq(CriterionReference("segment-age-one-bracket", 10)))
    )
  }

  scenario("user does not match criteria any more") {
    sendMessageAndVerifyOutput(
      input = UserPropertyDTO.PropertyType.Age(AgeDTO(75)),
      expectedOutput = CriteriaByType(
        CriterionType.AGE, 
        Seq.empty)
    )
  }
}
```
@[1]
@[2,4-7]
@[11,13-16]
Note: Mention:
1) Not only happy path 
2) Use buisness use-cases
---
### Component test using docker
  - Fast feedback (run any test locally)
  - Uses clean environment
  - Quick to run 
---
### Issues  
  - Dockerised AWS components may not be 100% identical to real ones
  - Mocked data in services, can be wrong assumptions
  - Not 100% similarity between environments (if isLocal)

---
### Best practices
  * Test one thing at a time 
  * Think, can the test be flaky?
  * Check not only one happy path, but business related feature
  
  
  <br />
  <br />
  
  * Don't check DB
  * Don't mock juice modules

  
  
Note: Example is age calculator that says that we should return an empty list if there is no age segments.
---
?
---
@vladimir.batygin
@xabier.laiseca
