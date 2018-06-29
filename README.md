# OneGuard Micro-Service Architecture Inter-Service Communication Protocol Specification

Specification of asynchronous Inter-Service Communication for OneGuard Micro-Service Architecture ecosystem.

## Architecture overview

Inter-service communication is facilitated via use of [AMQP](https://www.amqp.org/) broker. 
Specifically we use [RabbitMQ](http://www.rabbitmq.com/).

Each service instance is identified with a service name and instance ID.
**Service name** should be lower case words with removed punctuation and spaces replaced by single dash (`-`).
**Instance ID** must be a [UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier).

Each service instance subscribes to two queues - one specific to the instance
and one that is load-balanced between all instances of the service.

Service queue is bound to the exchange **`service.{service_name}`**,
and instance queue is bound to the exchange **`service.{service-name}.{instance-id}`**.

### Message format

Messages sent between service have standardized format. This is description of its properties:

- **`id`** `uuid` *mandatory* - message ID; recommend v4 UUID (randomly generated)
- **`type`** `string` *mandatory* - notification type - used to determine notification handler(s)
- **`principal`** `string` *optional* - security principal ID in whose authority the message is sent
- **`issuer`** `Issuer` *mandatory* - issuing service instance
  - **`service`** `string` *mandatory* - issuing instance service name
  - **`id`** `uuid` *mandatory* - issuing instance ID
- **`payload`** `object` *mandatory* - actual message content; structure depends on the specific message use case
- **`context`** `object` *optional* - 
    client context, to be returned unmodified on any response; structure depends solely on the issuer
- **`responseTo`** `uuid` *optional* - original message ID, this message was created as a result of, if any
- **`occurredAt`** `date` *mandatory* - 
    time of message occurrence in milliseconds since the epoch (1970-01-01 00:00:00.000 UTC)
- **`respondToInstance`** `boolean` *optional* - 
    if `true`, any response is to be sent to the specific instance that issued this message,
    otherwise it is to be sent to the service; defaults to `false`

Context is used by message issuer upon receiving the response.
For example it can be used to store the request ID for response matching.
                                       
AMQP message must be encoded in supported content type and must specify **`content_type`** property.
Currently only [JSON](https://www.json.org/) (content type **`application/json`**) is supported.
                                       
### Message example

Properties:

```json
{"content_type": "application/json"}
```

Content:

```json
{
  "id": "42944b91-a8df-4e88-8a62-98284496a67d",
  "type": "example.message",
  "principal": "73c656a2-51cb-4388-a23a-625e5bea3a67",
  "issuer": {
    "name": "example-service",
    "id": "c035a5f5-1db5-4bae-bd01-d6966b402f70"
  },
  "payload": {
    "greeting": "Hullo!"
  },
  "context": {
    "sessionId": "862b2a20-f1de-4617-abbc-72f5c9f7314b"
  },
  "responseTo": null,
  "occurredAt": 1514764800000,
  "respondToInstance": true
}
```

### Error handling

Following checks are executed on every message:

1) If **`content_type`** is not supported, the error is logged, and message is discarded.
1) If parsing fails for the specified content type, the error is logged, and message is discarded.
1) If mapping onto the model fails, the error is logged, an attempt to identify the issuer
   and report error to it is made, and message is discarded.
1) If an error occurs during message handling, it is reported to the issuer.

Error report messages have type `error.report` and their payload have following structure:

- **`message`** `string` *mandatory* - error message (description)
- **`errors`** `array` *mandatory* - array of Errors
  - **`Error`**
    - **`className`** `string` *mandatory*
    - **`message`** `string` *mandatory* - error message
    - **`stackTrace`** `array` *mandatory*
      - **`StackTraceElement`**
        - **`class`** `string` *optional* - 
            the fully qualified name of the class containing the execution point represented 
            by this stack trace element, or `null` if this information is unavailable
        - **`function`** `string` *mandatory* -
            the name of the function containing the execution point represented by the stack trace element
        - **`fileName`** `string` *optional* -
            the name of the file containing the execution point represented by the stack trace element,
            or `null` if this information is unavailable
        - **`lineNumber`** `int` *optional* -
            the line number of the source line containing the execution point represented by this stack trace element,
            or a negative number if this information is unavailable; defaults to `-1`

Error report message example:

```json
{
  "id": "message UUID",
  "type": "error.report",
  "principal": "user ID if applicable",
  "issuer": {
    "name": "reporting service name",
    "id": "reporting service instance ID"
  },
  "payload": {
    "message": "string error message",
    "errors": [
      {
        "type": "solutions.oneguard.msa.core.MessagingException",
        "message": "Could not convert the payload to 'String'",
        "stackTrace": [
          {
            "declaringClass": "org.example.ExampleHandler",
            "methodName": "handleMessage",
            "fileName": "ExampleHandler.java",
            "lineNumber": "13"
          }
        ],
        "cause": {
          "type": "java.lang.NullPointerException",
          "stackTrace": [
            {
              "declaringClass": "java.util.Objects",
              "methodName": "requireNonNull",
              "fileName": "Objects.java",
              "lineNumber": "221"
            },
            {
              "declaringClass": "org.example.ExampleHandler",
              "methodName": "convertPayload",
              "fileName": "ExampleHandler.java",
              "lineNumber": "23"
           }
          ]
        }
      }
    ]
  },
  "context": {"content": "replayed context"},
  "responseTo": "original message ID",
  "occurredAt": 1514764800000
}
```

### Submiting message from CLI

To submit the message to RabbitMQ directly from CLI, 
you need to have [`rabbitmqadmin`](https://www.rabbitmq.com/management-cli.html) installed.

The command below sends this minimal message to service `example-service`:

```json
{
  "id": "42944b91-a8df-4e88-8a62-98284496a67d",
  "type": "example.message",
  "issuer": {
    "name": "example-service",
    "id": "c035a5f5-1db5-4bae-bd01-d6966b402f70"
  },
  "payload": {},
  "occurredAt": 1514764800000
}
```

```bash
rabbitmqadmin publish \
    exchange=service.example-service \
    routing_key=anything \
    payload='{"id":"42944b91-a8df-4e88-8a62-98284496a67d","type":"example.message","issuer":{"name":"example-service","id":"c035a5f5-1db5-4bae-bd01-d6966b402f70"},"payload":{},"occurredAt":1514764800000}' \
    properties='{"content_type":"application/json"}'
```

### See also

- [OneGuard Micro-Service Architecture Core Library for Java](https://github.com/OneGuardSolutions/msa-core) -
    library for easy implementation of micro-services in Java that implements this protocol
- [RabbitMQ Clients & Developer Tools](https://www.rabbitmq.com/devtools.html) -
    documentation for varius RabbitMQ client libraries
