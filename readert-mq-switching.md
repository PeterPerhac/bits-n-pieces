### **✅ Test Setup That Switches Between IBM MQ & ActiveMQ**
You need a flexible test setup that:
1. **Uses Embedded ActiveMQ for local tests**.
2. **Uses a real IBM MQ instance when available**.
3. **Allows switching between them via configuration**.

---

## **🛠 Solution: Configuration-Based Broker Selection**
We'll use an **environment variable or config flag** to choose between **ActiveMQ (test) and IBM MQ (production-like integration tests)**.

### **1️⃣ Define a Trait for the MQ Connection Factory**
Create a trait that abstracts over the **JMS connection setup**:
```scala
import cats.effect._
import javax.jms._

trait MQConnectionProvider[F[_]] {
  def getConnectionFactory: Resource[F, ConnectionFactory]
}
```

---

### **2️⃣ Implement ActiveMQ Connection Provider (for Tests)**
This will be used when running **local tests**.
```scala
import org.apache.activemq.broker.BrokerService
import org.apache.activemq.ActiveMQConnectionFactory

object ActiveMQProvider extends MQConnectionProvider[IO] {
  private def startEmbeddedBroker: Resource[IO, BrokerService] =
    Resource.make {
      IO {
        val broker = new BrokerService()
        broker.setPersistent(false)
        broker.addConnector("vm://localhost")
        broker.start()
        broker
      }
    } { broker =>
      IO(broker.stop()).handleErrorWith(_ => IO.unit) // Ensure cleanup
    }

  override def getConnectionFactory: Resource[IO, ConnectionFactory] =
    startEmbeddedBroker *> Resource.pure(new ActiveMQConnectionFactory("vm://localhost"))
}
```
- **Starts an embedded ActiveMQ broker** for local testing.
- **Cleans up automatically** when tests finish.

---

### **3️⃣ Implement IBM MQ Connection Provider (for Real Tests)**
This connects to a **real IBM MQ broker**.
```scala
import com.ibm.mq.jms.MQQueueConnectionFactory
import com.ibm.msg.client.wmq.common.CommonConstants._

object IBMMQProvider extends MQConnectionProvider[IO] {
  override def getConnectionFactory: Resource[IO, ConnectionFactory] =
    Resource.pure {
      val factory = new MQQueueConnectionFactory()
      factory.setHostName("ibm-mq-host")
      factory.setPort(1414)
      factory.setQueueManager("QM1")
      factory.setChannel("DEV.ADMIN.SVRCONN")
      factory.setTransportType(WMQ_CM_CLIENT)
      factory
    }
}
```
- **Connects to IBM MQ** using real credentials.
- **Configurable host, port, queue manager, and channel**.

---

### **4️⃣ Create a Configurable Test Setup**
This **chooses the right MQ provider** based on an environment variable.

```scala
object MQTestSetup {
  def getProvider: MQConnectionProvider[IO] = sys.env.get("USE_IBMMQ") match {
    case Some("true") => IBMMQProvider  // Use IBM MQ
    case _            => ActiveMQProvider // Default to ActiveMQ for local tests
  }

  def getConnectionFactory: Resource[IO, ConnectionFactory] = getProvider.getConnectionFactory
}
```
- **Reads `USE_IBMMQ` from environment variables**.
- **Defaults to ActiveMQ** when not set.

---

### **5️⃣ Use This in Your Tests**
Now, tests can **automatically use the correct MQ provider**.

```scala
import munit.CatsEffectSuite
import javax.jms._

class MQIntegrationTest extends CatsEffectSuite {

  test("JMS should send and receive messages") {
    MQTestSetup.getConnectionFactory.use { connectionFactory =>
      IO {
        val connection = connectionFactory.createConnection()
        val session = connection.createSession(false, Session.CLIENT_ACKNOWLEDGE)
        val queue = session.createQueue("TEST.QUEUE")

        // ✅ Send a message
        val producer = session.createProducer(queue)
        producer.send(session.createTextMessage("Hello, JMS!"))

        // ✅ Receive the message
        val consumer = session.createConsumer(queue)
        connection.start()
        val receivedMsg = consumer.receive(5000).asInstanceOf[TextMessage]
        receivedMsg.acknowledge()

        // ✅ Verify the content
        assertEquals(receivedMsg.getText, "Hello, JMS!")
      }
    }
  }
}
```
- Runs on **ActiveMQ by default**.
- Set **`USE_IBMMQ=true`** to switch to IBM MQ.

---

## **🚀 Running Tests**
- **Run with ActiveMQ (default)**
  ```bash
  sbt test
  ```
- **Run with IBM MQ**
  ```bash
  USE_IBMMQ=true sbt test
  ```

---

## **🔥 Final Thoughts**
| **Feature** | **Implementation** |
|------------|------------------|
| **Uses ActiveMQ for local tests** | Embedded ActiveMQ (`BrokerService`) |
| **Uses IBM MQ when configured** | Reads `USE_IBMMQ` env var |
| **Ensures proper cleanup** | Uses `Resource[IO, ConnectionFactory]` |
| **Easily switchable** | Just set `USE_IBMMQ=true` |

Would you like **a Docker-based IBM MQ setup** for local testing as well? 🚀


### **✅ Using the `Reader` Pattern to Switch Between IBM MQ & ActiveMQ**
Since your app already uses:
- **`Reader[MQConfig, MqAdapter]`** to instantiate an MQ adapter based on `MQConfig`
- You want to **use IBM MQ in production** and **Embedded ActiveMQ in tests**

We can make `Reader` resolve the appropriate `MqAdapter` **dynamically** based on the `MQConfig`.

---

## **🛠 1️⃣ Define `MQConfig` for IBM MQ and ActiveMQ**
Since **IBM MQ and ActiveMQ require different configurations**, we define `MQConfig` as:

```scala
sealed trait MQConfig
case class ActiveMQConfig(brokerUrl: String) extends MQConfig
case class IBMMQConfig(host: String, port: Int, queueManager: String, channel: String) extends MQConfig
```
- `ActiveMQConfig` → Used in **tests** (connects to an **embedded ActiveMQ**).
- `IBMMQConfig` → Used in **production** (connects to a **real IBM MQ**).

---

## **🛠 2️⃣ Define `MqAdapter` (The Consumer)**
```scala
trait MqAdapter {
  def consumeMessages: fs2.Stream[IO, TextMessage]
}
```
Each implementation will **pull messages from the correct broker**.

---

## **🛠 3️⃣ Implement an MQ Consumer for IBM MQ**
This version **connects to a real IBM MQ broker**.
```scala
import javax.jms._
import cats.effect._
import fs2._
import com.ibm.mq.jms.MQQueueConnectionFactory
import com.ibm.msg.client.wmq.common.CommonConstants._

class IBMMQAdapter(config: IBMMQConfig) extends MqAdapter {
  private def createConnectionFactory: ConnectionFactory = {
    val factory = new MQQueueConnectionFactory()
    factory.setHostName(config.host)
    factory.setPort(config.port)
    factory.setQueueManager(config.queueManager)
    factory.setChannel(config.channel)
    factory.setTransportType(WMQ_CM_CLIENT)
    factory
  }

  def consumeMessages: Stream[IO, TextMessage] = {
    def connect: IO[(Connection, Session, MessageConsumer)] =
      IO {
        val connection = createConnectionFactory.createConnection()
        connection.start()
        val session = connection.createSession(false, Session.CLIENT_ACKNOWLEDGE)
        val queue = session.createQueue("PROD.QUEUE")
        val consumer = session.createConsumer(queue)
        (connection, session, consumer)
      }

    Stream.resource(Resource.make(connect) { case (conn, session, consumer) =>
      IO(consumer.close()) *> IO(session.close()) *> IO(conn.close())
    }).flatMap { case (_, _, consumer) =>
      Stream.repeatEval(IO(consumer.receive().asInstanceOf[TextMessage]))
    }
  }
}
```
- **Creates a real IBM MQ connection**.
- **Pulls messages as an fs2 Stream**.
- **Cleans up the connection** when done.

---

## **🛠 4️⃣ Implement an MQ Consumer for Embedded ActiveMQ (Testing)**
This version **uses ActiveMQ’s in-memory broker**.

```scala
import org.apache.activemq.ActiveMQConnectionFactory
import org.apache.activemq.broker.BrokerService

class ActiveMQAdapter(config: ActiveMQConfig) extends MqAdapter {
  private def createConnectionFactory: ConnectionFactory =
    new ActiveMQConnectionFactory(config.brokerUrl)

  def consumeMessages: Stream[IO, TextMessage] = {
    def connect: IO[(Connection, Session, MessageConsumer)] =
      IO {
        val connection = createConnectionFactory.createConnection()
        connection.start()
        val session = connection.createSession(false, Session.CLIENT_ACKNOWLEDGE)
        val queue = session.createQueue("TEST.QUEUE")
        val consumer = session.createConsumer(queue)
        (connection, session, consumer)
      }

    Stream.resource(Resource.make(connect) { case (conn, session, consumer) =>
      IO(consumer.close()) *> IO(session.close()) *> IO(conn.close())
    }).flatMap { case (_, _, consumer) =>
      Stream.repeatEval(IO(consumer.receive().asInstanceOf[TextMessage]))
    }
  }
}
```
- Uses **ActiveMQ’s in-memory broker** (for tests).
- **Pulls messages** like the IBM MQ version.
- **Fully interchangeable** with `IBMMQAdapter`.

---

## **🛠 5️⃣ Implement `Reader[MQConfig, MqAdapter]`**
Now we use `Reader` to **choose the correct MQ Adapter based on `MQConfig`**:

```scala
import cats.data.Reader

val mqAdapterReader: Reader[MQConfig, MqAdapter] = Reader {
  case cfg: IBMMQConfig  => new IBMMQAdapter(cfg)
  case cfg: ActiveMQConfig => new ActiveMQAdapter(cfg)
}
```
- If **given `IBMMQConfig`**, it returns **`IBMMQAdapter`**.
- If **given `ActiveMQConfig`**, it returns **`ActiveMQAdapter`**.

---

## **🛠 6️⃣ Using It in Production & Tests**
### **Production (IBM MQ)**
```scala
val prodConfig = IBMMQConfig(
  host = "ibm-mq-host",
  port = 1414,
  queueManager = "QM1",
  channel = "DEV.ADMIN.SVRCONN"
)

val prodMqAdapter: MqAdapter = mqAdapterReader.run(prodConfig)

prodMqAdapter.consumeMessages
  .evalMap(msg => IO(println(s"Received: ${msg.getText}")))
  .compile
  .drain
  .unsafeRunSync()
```
- **Runs against IBM MQ**.

---

### **Testing (ActiveMQ)**
```scala
val testConfig = ActiveMQConfig("vm://localhost")

val testMqAdapter: MqAdapter = mqAdapterReader.run(testConfig)

testMqAdapter.consumeMessages
  .evalMap(msg => IO(println(s"Test Received: ${msg.getText}")))
  .compile
  .drain
  .unsafeRunSync()
```
- **Runs against Embedded ActiveMQ**.

---

## **🔥 Benefits of Using `Reader[MQConfig, MqAdapter]`**
| **Feature** | **Benefit** |
|------------|------------|
| **No `if/else` logic** | `Reader` dynamically resolves the correct MQ adapter. |
| **Works for IBM MQ & ActiveMQ** | Can seamlessly switch between **test and production** setups. |
| **Easily testable** | Pass in **ActiveMQConfig** for local testing. |
| **Clean separation of concerns** | The **consumer logic** is **decoupled from MQ selection**. |

---

## **🚀 Next Steps**
1. **Do you need logging** to track reconnections in case MQ fails?
2. **Would you like a `ReaderT[IO, MQConfig, Stream[IO, TextMessage]]` approach** to handle effects better?
3. **Would you like a test suite to verify switching between IBM MQ & ActiveMQ?**

Let me know how I can refine this further! 🚀


### **✅ Enhancing the MQ Consumer with Logging and `ReaderT` for Better Effect Handling**
Since you want:
1. **Logging to track MQ reconnections** when failures occur.
2. **A `ReaderT[IO, MQConfig, Stream[IO, TextMessage]]` approach** to better handle effects.

We'll refactor the existing `Reader[MQConfig, MqAdapter]` pattern into `ReaderT[IO, MQConfig, Stream[IO, TextMessage]]`.

---

## **🔹 1️⃣ Why Use `ReaderT[IO, MQConfig, Stream[IO, TextMessage]]`?**
Instead of returning an **`MqAdapter`**, we directly return an `fs2.Stream[IO, TextMessage]`, meaning:
- **No need to instantiate an adapter object**.
- **Directly returns an effectful stream** that reacts to MQConfig.
- **Handles MQ failures & reconnections transparently**.

---

## **🛠 2️⃣ Implement `ReaderT` for Dynamic MQ Selection**
```scala
import cats.data.ReaderT
import cats.effect._
import fs2._
import javax.jms._
import scala.concurrent.duration._
import org.typelevel.log4cats.LoggerFactory
import org.typelevel.log4cats.slf4j.Slf4jLogger

// MQConfig Definitions
sealed trait MQConfig
case class ActiveMQConfig(brokerUrl: String) extends MQConfig
case class IBMMQConfig(host: String, port: Int, queueManager: String, channel: String) extends MQConfig

object MQConsumer {

  // 🔹 Logger for tracking reconnections
  implicit def logger[F[_]: Sync] = Slf4jLogger.getLogger[F]

  // 🔹 ReaderT for MQ Consumer that transparently handles reconnections
  val mqStreamReader: ReaderT[IO, MQConfig, Stream[IO, TextMessage]] =
    ReaderT { config =>
      config match {
        case cfg: IBMMQConfig   => createResilientMQConsumer(cfg)
        case cfg: ActiveMQConfig => createResilientActiveMQConsumer(cfg)
      }
    }

  // 🔹 IBM MQ Consumer with Automatic Reconnection
  private def createResilientMQConsumer(cfg: IBMMQConfig): Stream[IO, TextMessage] = {
    def connect: IO[(Connection, Session, MessageConsumer)] =
      IO {
        val factory = new com.ibm.mq.jms.MQQueueConnectionFactory()
        factory.setHostName(cfg.host)
        factory.setPort(cfg.port)
        factory.setQueueManager(cfg.queueManager)
        factory.setChannel(cfg.channel)
        factory.setTransportType(com.ibm.msg.client.wmq.common.CommonConstants.WMQ_CM_CLIENT)

        val connection = factory.createConnection()
        connection.start()
        val session = connection.createSession(false, Session.CLIENT_ACKNOWLEDGE)
        val queue = session.createQueue("PROD.QUEUE")
        val consumer = session.createConsumer(queue)
        (connection, session, consumer)
      }

    def consumeMessages(consumer: MessageConsumer): Stream[IO, TextMessage] =
      Stream.repeatEval(IO(consumer.receive().asInstanceOf[TextMessage]))

    // 🔹 Auto-reconnect logic
    Stream.resource(Resource.make(connect) { case (conn, session, consumer) =>
      IO(consumer.close()) *> IO(session.close()) *> IO(conn.close())
    }).flatMap { case (_, _, consumer) =>
      consumeMessages(consumer)
    }.handleErrorWith { err =>
      Stream.eval(logger[IO].error(s"IBM MQ Consumer failed: ${err.getMessage}, reconnecting...")) >>
        Stream.sleep_[IO](5.seconds) >> // Delay before retrying
        createResilientMQConsumer(cfg)
    }
  }

  // 🔹 ActiveMQ Consumer with Auto-Reconnection (for Tests)
  private def createResilientActiveMQConsumer(cfg: ActiveMQConfig): Stream[IO, TextMessage] = {
    def connect: IO[(Connection, Session, MessageConsumer)] =
      IO {
        val factory = new org.apache.activemq.ActiveMQConnectionFactory(cfg.brokerUrl)
        val connection = factory.createConnection()
        connection.start()
        val session = connection.createSession(false, Session.CLIENT_ACKNOWLEDGE)
        val queue = session.createQueue("TEST.QUEUE")
        val consumer = session.createConsumer(queue)
        (connection, session, consumer)
      }

    def consumeMessages(consumer: MessageConsumer): Stream[IO, TextMessage] =
      Stream.repeatEval(IO(consumer.receive().asInstanceOf[TextMessage]))

    // 🔹 Auto-reconnect logic
    Stream.resource(Resource.make(connect) { case (conn, session, consumer) =>
      IO(consumer.close()) *> IO(session.close()) *> IO(conn.close())
    }).flatMap { case (_, _, consumer) =>
      consumeMessages(consumer)
    }.handleErrorWith { err =>
      Stream.eval(logger[IO].error(s"ActiveMQ Consumer failed: ${err.getMessage}, reconnecting...")) >>
        Stream.sleep_[IO](5.seconds) >>
        createResilientActiveMQConsumer(cfg)
    }
  }
}
```

---

## **🛠 3️⃣ Using the `ReaderT`-based MQ Consumer**
Since `mqStreamReader` is now a **`ReaderT[IO, MQConfig, Stream[IO, TextMessage]]`**, you can use it as follows:

### **✅ Production (IBM MQ)**
```scala
val prodConfig = IBMMQConfig(
  host = "ibm-mq-host",
  port = 1414,
  queueManager = "QM1",
  channel = "DEV.ADMIN.SVRCONN"
)

val prodStream: Stream[IO, TextMessage] = MQConsumer.mqStreamReader.run(prodConfig)

prodStream
  .evalMap(msg => IO(println(s"Received: ${msg.getText}")))
  .compile
  .drain
  .unsafeRunSync()
```

### **✅ Testing (ActiveMQ)**
```scala
val testConfig = ActiveMQConfig("vm://localhost")

val testStream: Stream[IO, TextMessage] = MQConsumer.mqStreamReader.run(testConfig)

testStream
  .evalMap(msg => IO(println(s"Test Received: ${msg.getText}")))
  .compile
  .drain
  .unsafeRunSync()
```

---

## **🔥 Key Improvements**
| **Feature** | **Benefit** |
|------------|------------|
| **Uses `ReaderT` for cleaner dependency injection** | No manual object creation needed. |
| **Directly returns `Stream[IO, TextMessage]`** | No extra adapter objects, just the stream. |
| **Handles connection failures transparently** | Logs failures and retries automatically. |
| **Supports both IBM MQ (production) & ActiveMQ (tests)** | Easily switch MQ providers. |

---

## **🚀 Next Steps**
Would you like:
1. **Configurable logging levels (info/debug/error) for easier debugging?**
2. **Structured logging (JSON logs) for production monitoring?**

Let me know how far you want to optimize this! 🚀
