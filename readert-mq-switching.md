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
