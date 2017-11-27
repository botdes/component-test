@title[Introduction]

# Component test in ATE

---
@title[Plan]
## Plan

* What is ATE
* Component in testing pyramid
* Evaluation of component test in ATE
* Best practices
---

@title[What is ATE]

## What is ATE?
Note: diagram here pulse, um, users -> ATE -> users/segments
to undertand the scope of the problem.
---

@title[What is ATE]

## What is ATE?
#### 15 backend engeneers
#### 63 gihub repos
#### Ec2, Kineis, SQS, Postgres, Aerospike

---

## Component test in testing pyramid

Note:
Add a picture here:
Introduced fairly recently

---
## What requirements for component test
* Close to production
* Blackbox 
* Fast
* Running locally

Note: 

---

## Initial state
### Only Unit, Integration and Smoke Tests
* Test manually via datadog in dev and pre environments
* Issues in juice in dev
* Long feedback loop 
* Shared AWS resources between tests / lots of AWS garbage
* Not very stable
* Issues with local run / security 
* Slow

---
## Setup
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

  private val kinesisContainerHolder =
    registerContainerRunner(
      KinesisContainerDetailsBuilder.withDynamicPort().get.build()
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
    val kinesisClient = kinesisContainerHolder.container.newClient()
    createStream(kinesisClient, kinesisInputStreamName)
    createStream(kinesisClient, kinesisOutputStreamName)
    sys.props += "spt.kinesis.consumer.streamName" -> kinesisInputStreamName
    sys.props += "spt.kinesis.publisher.streamName" -> kinesisOutputStreamName
    sys.props += "spt.kinesis.endpoint" -> kinesisContainerHolder.container.endpoint
    sys.props += "spt.dynamodb.endpoint" -> dynamoContainerHolder.container.endpoint
  }

  private def createStream(kinesisClient: AmazonKinesis, kinesisStreamName: String): Unit = {
    kinesisClient.createStream(kinesisStreamName, 1)
    (1 to 120).toStream
      .map(_ => kinesisClient.describeStream(kinesisStreamName).getStreamDescription.getStreamStatus)
      .takeWhile(_ != "ACTIVE").last
  }

```
@[6]
@[18-20]
@[32,36,41]
@[43-50]
@[32,40-41]


---
## 

---

##
1) what is ATE
    - backed, 20 + micro services (http/sqs/kinesis)
2) component in testing piramind
    - unit/ integration / component / smoke test 
3) what requirements for component test
    - close to production
    - blackbox 
    - fast
    - running locally
4) Initial state: No component test:
    - test manually via datadog in dev / pre
    - failures in dev with juice 
    - very long feedback loop
5) Intermediate state: integration test using real AWS
    - slow
    - shared resources between tests
    - not very stable
    - leave lots of garbage in aws
    - issues with local run / security 
6) Component test using docker
    - fast
    - close to prod
    - nearly blackbox
    - not 100% similarity between environments (if isLocal)
    - mocked data in services, can be wrong assumptions
7) Component test best practices
    - use as blackbox, don't' check the db
    - don't mock "inconvenient" classes
    - test one thing 
    - check not only one happy path, but business related feature.
