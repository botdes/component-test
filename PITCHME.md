@title[Introduction]

# ATE Component Tests
Note:
My is Vladimir Batygin, I'm techlead in ATE team. Today I want to share with you how we do component test in ATE.
---
@title[Plan]
## Plan

* Component in testing pyramid
* Evaluation of component test in ATE
* How to
* Dos and Don'ts
Note: 
---
#### Where will you place component test?
![test pyramid](p1_2.png)

---
#### Component test
![test pyramid](p2_2.png)

---
* Unit test - test only one class
* Integration - test external dependecies
* <b>Component - test the whole microservice</b>
* Smoke - verify end to end

---
## Requirements for component tests
* Close to production
* Running locally
* Test all buisness requirements

---
# 1.5 years ago
#### Integration test using developers account in AWS
#### Run manually smoke test
#### Look at datadog metrics in dev and pre environments

---
## Issues
* Exceptions on startup (not the whole code is tested)
* Issues with local run and security 
* Slow feedback loop
* Not ready for Continious Deployment

---
## Solution
* Ligtweigt test that tests the whole application
* Use dockerised AWS
---

### Shared docker lib

[shared-docker-test-containers](https://github.schibsted.io/spt-advertising/shared-docker-test-containers)
<br />
<br />
based on 
`com.spotify:docker-client`

---
## Example of a microservice

![calculator](calc.png)
---
## Create a test in 3 steps

---
## Step 1 - Setup Kinesis
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
@[6](Mix in shared test dependencies)
@[18-20] (Create containters)
@[33,37,42] (Intialise kinisis)
@[45-47] (Create streams)
@[49-51] (Update config parameters)
---
## Step 2 - Stup Segment Definition
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
## Step 3 - Tests 
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
### Benefits of dockerised tests
  - Fast feedback (run any test locally)
  - Uses clean environment
  - Can be reproduced on any machine
---
### Areas of improvement  
  - Dockerised AWS components may not be 100% identical to real ones
  - Not 100% similarity between environments 
  - Mocked data in services, can be wrong assumptions
---
### DOs
  * Test all business related features
  * Test multiple interractions
  * Think, can the test be flaky?
---
### Don'ts
  * Don't validate intermediate state (DB for example)
  * Don't use mocks
  
Note: Example is age calculator that says that we should return an empty list if there are no age segments.
---
# Thank you
