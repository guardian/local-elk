# Local ELK

Run small [ELK stack](https://www.elastic.co/elk-stack) locally within a Docker environment.

Largely based on [docker-elk](https://github.com/deviantony/docker-elk) but without the xpack configuration, as such this is:

**FOR LOCAL USE ONLY**

## Why?
An ELK stack provides a simple way to analyse application logs. We do this in PROD.

In PROD, we typically use the [kinesis logback appender](https://github.com/guardian/kinesis-logback-appender) to ship logs to ELK.

Locally, we typically write logs to disk. Whilst we can `tail` these to see them, it isn't very easy. 
This becomes more obvious when we start to log with markers.

Markers add structure to logs and allow us to group logs together. 
For example, in a REST API, if we log the `method` as a marker we can easily search for `DELETE` requests.

## Usage
- `./script/setup`
- `./script/start`
- Open `https://logs.local.dev-gutools.co.uk` in the browser to access Kibana
- Ship logs from your application 

## Shipping logs from your application
local-elk accepts tcp input on port 5000 in the JSON codec format. 
We can use the `LogstashTcpSocketAppender` to ship logs to it from our application.

TIP: You'll likely want to have a guard to only use the `LogstashTcpSocketAppender` when in DEV and when you know local-elk is running.

Examples assume you are using the [Play!](https://www.playframework.com/) framework in Scala.

### Via logback.xml
Add an appender to your logback configuration file:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
    <destination>127.0.0.1:5000</destination>
    <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
  </appender>
        
  <root level="INFO">
    <appender-ref ref="LOGSTASH"/>
  </root>
</configuration>
```

More information [here](https://github.com/logstash/logstash-logback-encoder#tcp-appenders).

### Via application code
While shipping logs via the logback configuration is fine, we may want to augment logs with more information prior to writing them. 
For example, in PROD we'd add the `Stage`, `Stack` and `App` tags.

Within Play! logging is usually setup in the `ApplicationLoader`.

To ship logs to local-elk we need to add a `LogstashTcpSocketAppender` to the `Logger`:

```scala
// In AppLoader.scala
import play.api.ApplicationLoader.Context
import play.api.{Application, ApplicationLoader}

class AppLoader extends ApplicationLoader {
  final override def load(context: Context): Application = {
    LogConfig.initLocalLogShipping
  }
}

// In LogConfig.scala
import ch.qos.logback.classic.{LoggerContext, Logger => LogbackLogger}
import net.logstash.logback.appender.LogstashTcpSocketAppender
import net.logstash.logback.encoder.LogstashEncoder
import org.slf4j.{LoggerFactory, Logger => SLFLogger}
import java.net.InetSocketAddress
import play.api.libs.json._
import scala.util.Try

object LogConfig {
  private val rootLogger: LogbackLogger = LoggerFactory.getLogger(SLFLogger.ROOT_LOGGER_NAME).asInstanceOf[LogbackLogger]
  private val BUFFER_SIZE = 1000

  private def createCustomFields(): String = {
    Json.toJson(Map(
        "stack" -> "local-elk",
        "stage" -> "DEV",
        "app"     -> "demo"
    )).toString()
  }

  private def createLogstashAppender(context: LoggerContext): LogstashTcpSocketAppender = {
    val customFields = createCustomFields()
    
    val appender = new LogstashTcpSocketAppender()

    appender.setContext(context)
    appender.addDestinations(new InetSocketAddress("localhost", 5000))
    appender.setWriteBufferSize(BUFFER_SIZE)
        
    val encoder = new LogstashEncoder()
    encoder.setCustomFields(customFields)
    encoder.start()

    appender.setEncoder(encoder)
    appender.start()

    appender
  }

  def initLocalLogShipping: Unit = {
    Try {
      rootLogger.info("Configuring local logstash log shipping")
      val appender = createLogstashAppender(rootLogger.getLoggerContext)
      rootLogger.addAppender(appender)
      rootLogger.info("Local logstash log shipping configured")
    } recover {
      case e => rootLogger.error("LogConfig Failed!", e)
    }
  }
}
```

### Alternatives
We could use the [File input plugin](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-file.html) to ship logs from disk.

However, this has a couple of problems:
- The files need to be mounted into the Logstash container of local-elk. This doesn't scale well and isn't very generic.
- The log files on disk need to be in the `net.logstash.logback.encoder.LogstashEncoder` encoder to create JSON data. 
This is less human readable, in the event that reading these files becomes necessary.
