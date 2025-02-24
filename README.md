# NestJS - RabbitMQ custom strategy

![alt cover](https://github.com/AlariCode/nestjs-rmq/raw/master/img/new-logo.jpg)

**More NestJS libs on [alariblog.ru](https://alariblog.ru)**

[![npm version](https://badgen.net/npm/v/nestjs-rmq)](https://www.npmjs.com/package/nestjs-rmq)
[![npm version](https://badgen.net/npm/license/nestjs-rmq)](https://www.npmjs.com/package/nestjs-rmq)
[![npm version](https://badgen.net/github/open-issues/AlariCode/nestjs-rmq)](https://github.com/AlariCode/nestjs-rmq/issues)
[![npm version](https://badgen.net/github/prs/AlariCode/nestjs-rmq)](https://github.com/AlariCode/nestjs-rmq/pulls)

This library will take care of RPC requests and messaging between microservices. It is easy to bind to our existing controllers to RMQ routes. This version is only for NestJS.

**Updated for NestJS 8!**

## Why use this over RabbitMQ transport in NestJS docs?

- Support for RMQ queue patterns with * and #.
- Using exchanges with topic bindings rather the direct queue sending.
- Additional `forTest()` method for emulating messages in unit or e2e tests without needing of RabbitMQ instance.
- Additional decorators for getting info out of messages.
- Support for class-validator decorators.
- Real production usage with more than 100 microservices.

## Start

First, install the package:

```bash
npm i nestjs-rmq
```

Setup your connection in root module:

```typescript
import { RMQModule } from 'nestjs-rmq';

@Module({
	imports: [
		RMQModule.forRoot({
			exchangeName: configService.get('AMQP_EXCHANGE'),
			connections: [
				{
					login: configService.get('AMQP_LOGIN'),
					password: configService.get('AMQP_PASSWORD'),
					host: configService.get('AMQP_HOST'),
				},
			],
		}),
	],
})
export class AppModule {}
```

In forRoot() you pass connection options:

-   **exchangeName** (string) - Exchange that will be used to send messages to.
-   **connections** (Object[]) - Array of connection parameters. You can use RMQ cluster by using multiple connections.

Additionally, you can use optional parameters:

-   **queueName** (string) - Queue name which your microservice would listen and bind topics specified in '@RMQRoute' decorator to this queue. If this parameter is not specified, your microservice could send messages and listen to reply or send notifications, but it couldn't get messages or notifications from other services. If you use empty string, RabbitMQ will generate name for you.
    Example:

```typescript
{
	exchangeName: 'my_exchange',
	connections: [
		{
			login: 'admin',
			password: 'admin',
			host: 'localhost',
		},
	],
	queueName: 'my-service-queue',
}
```
-   **connectionOptions** (object) - Additional connection options. You can read more [here](http://www.squaremobius.net/amqp.node/).
-   **prefetchCount** (boolean) - You can read more [here](http://www.squaremobius.net/amqp.node/).
-   **isGlobalPrefetchCount** (boolean) - You can read more [here](http://www.squaremobius.net/amqp.node/).
-	**queueOptions** (object) - options for created queue.
-   **reconnectTimeInSeconds** (number) - Time in seconds before reconnection retry. Default is 5 seconds.
-   **heartbeatIntervalInSeconds** (number) - Interval to send heartbeats to broker. Defaults to 5 seconds.
-   **queueArguments** (!!! deprecated. Use queueOptions instead) - You can read more about queue parameters [here](https://www.rabbitmq.com/parameters.html).
-   **messagesTimeout** (number) - Number of milliseconds 'post' method will wait for the response before a timeout error. Default is 30 000.
-   **isQueueDurable** (!!! deprecated. Use queueOptions instead) - Makes created queue durable. Default is true.
-   **isExchangeDurable** (!!! deprecated. Use exchangeOptions instead) - Makes created exchange durable. Default is true.
-   **exchangeOptions** (Options.AssertExchange) - You can read more about exchange options [here](squaremobius.net/amqp.node/channel_api.html#channel_assertExchange).
-   **logMessages** (boolean) - Enable printing all sent and recieved messages in console with its route and content. Default is false.
-   **logger** (LoggerService) - Your custom logger service that implements `LoggerService` interface. Compatible with Winston and other loggers.
-   **middleware** (array) - Array of middleware functions that extends `RMQPipeClass` with one method `transform`. They will be triggered right after recieving message, before pipes and controller method. Trigger order is equal to array order.
-   **errorHandler** (class) - custom error handler for dealing with errors from replies, use `errorHandler` in module options and pass class that extends `RMQErrorHandler`.
-   **serviceName** (string) - service name for debugging.

```typescript
class LogMiddleware extends RMQPipeClass {
	async transfrom(msg: Message): Promise<Message> {
		console.log(msg);
		return msg;
	}
}
```

-   **intercepters** (array) - Array of intercepter functions that extends `RMQIntercepterClass` with one method `intercept`. They will be triggered before replying on any message. Trigger order is equal to array order.

```typescript
export class MyIntercepter extends RMQIntercepterClass {
	async intercept(res: any, msg: Message, error: Error): Promise<any> {
		// res - response body
		// msg - initial message we are replying to
		// error - error if exists or null
		return res;
	}
}
```

Config example with middleware and intercepters:

```typescript
import { RMQModule } from 'nestjs-rmq';

@Module({
	imports: [
		RMQModule.forRoot({
			exchangeName: configService.get('AMQP_EXCHANGE'),
			connections: [
				{
					login: configService.get('AMQP_LOGIN'),
					password: configService.get('AMQP_PASSWORD'),
					host: configService.get('AMQP_HOST'),
				},
			],
			middleware: [LogMiddleware],
			intercepters: [MyIntercepter],
		}),
	],
})
export class AppModule {}
```

## Async initialization

If you want to inject dependency into RMQ initialization like Configuration service, use `forRootAsync`:

```typescript
import { RMQModule } from 'nestjs-rmq';
import { ConfigModule } from './config/config.module';
import { ConfigService } from './config/config.service';

@Module({
	imports: [
		RMQModule.forRootAsync({
			imports: [ConfigModule],
			inject: [ConfigService],
			useFactory: (configService: ConfigService) => {
				return {
					exchangeName: 'test',
					connections: [
						{
							login: 'guest',
							password: 'guest',
							host: configService.getHost(),
						},
					],
					queueName: 'test',
				};
			},
		}),
	],
})
export class AppModule {}
```

-   **useFactory** - returns `IRMQServiceOptions`.
-   **imports** - additional modules for configuration.
-   **inject** - additional services for usage inside useFactory.

## Sending messages

To send message with RPC topic use send() method in your controller or service:

```typescript
@Injectable()
export class ProxyUpdaterService {
	constructor(private readonly rmqService: RMQService) {}

	myMethod() {
		this.rmqService.send<number[], number>('sum.rpc', [1, 2, 3]);
	}
}
```

This method returns a Promise. First type - is a type you send, and the second - you recive.

-   'sum.rpc' - name of subscription topic that you are sending to.
-   [1, 2, 3] - data payload.
    To get a reply:

```typescript
this.rmqService.send<number[], number>('sum.rpc', [1, 2, 3])
    .then(reply => {
        //...
    })
    .catch(error: RMQError => {
        //...
    });
```

Also you can use send options:

```typescript
this.rmqService.send<number[], number>('sum.rpc', [1, 2, 3], {
	expiration: 1000,
	priority: 1,
	persistent: true,
	timeout: 30000,
});
```

-   **expiration** - if supplied, the message will be discarded from a queue once it’s been there longer than the given number of milliseconds.
-   **priority** - a priority for the message.
-   **persistent** - if truthy, the message will survive broker restarts provided it’s in a queue that also survives restarts.
-   **timeout** - if supplied, the message will have its own timeout.

If you want to just notify services:

```typescript
const a = this.rmqService.notify<string>('info.none', 'My data');
```

This method returns a Promise.

-   'info.none' - name of subscription topic that you are notifying.
-   'My data' - data payload.

## Recieving messages

To listen for messages bind your controller or service methods to subscription topics with **RMQRoute()** decorator:

```typescript
export class AppController {
	//...

	@RMQRoute('sum.rpc')
	sum(numbers: number[]): number {
		return numbers.reduce((a, b) => a + b, 0);
	}

	@RMQRoute('info.none')
	info(data: string) {
		console.log(data);
	}
}
```

Return value will be send back as a reply in RPC topic. In 'sum.rpc' example it will send sum of array values. And sender will get `6`:

```typescript
this.rmqService.send('sum.rpc', [1, 2, 3]).then((reply) => {
	// reply: 6
});
```

Each '@RMQRoute' topic will be automatically bound to queue specified in 'queueName' option. If you want to return an Error just throw it in your method. To set '-x-status-code' use custom RMQError class.

```typescript
@RMQRoute('my.rpc')
myMethod(numbers: number[]): number {
	//...
    throw new RMQError('Error message', 2);
	throw new Error('Error message');
	//...
}
```

## Message patterns

With exchange type `topic` you can use message patterns to subscribe to messages that corresponds to that pattern. You can use special symbols:
- `*` - (star) can substitute for exactly one word.
- `#`- (hash) can substitute for zero or more words.

For example:
- Pattern `*.*.rpc` will match `my.own.rpc` or `any.other.rpc` and will not match `this.is.cool.rpc` or `my.rpc`.
- Pattern `compute.#` will match `compute.this.equation.rpc` and will not `do.compute.anything`.

To subscribe to pattern, use it as route:

```typescript
import { RMQRoute } from 'nestjs-rmq';

@RMQRoute('*.*.rpc')
myMethod(): number {
	// ...
}
```

> Note: If two routes patterns matches message topic, only the first will be used.

## Getting message metadata

To get more information from message (not just content) you can use `@RMQMessage` parameter decorator:

```typescript
import { RMQRoute, Validate, RMQMessage, ExtendedMessage } from 'nestjs-rmq';

@RMQRoute('my.rpc')
myMethod(data: myClass, @RMQMessage msg: ExtendedMessage): number {
	// ...
}
```

You can get all message properties that RMQ gets. Example:

```json
{
	"fields": {
		"consumerTag": "amq.ctag-1CtiEOM8ioNFv-bzbOIrGg",
		"deliveryTag": 2,
		"redelivered": false,
		"exchange": "test",
		"routingKey": "appid.rpc"
	},
	"properties": {
		"contentType": "undefined",
		"contentEncoding": "undefined",
		"headers": {},
		"deliveryMode": "undefined",
		"priority": "undefined",
		"correlationId": "ce7df8c5-913c-2808-c6c2-e57cfaba0296",
		"replyTo": "amq.rabbitmq.reply-to.g2dkABNyYWJiaXRAOTE4N2MzYWMyM2M0AAAenQAAAAAD.bDT8S9ZIl5o3TGjByqeh5g==",
		"expiration": "undefined",
		"messageId": "undefined",
		"timestamp": "undefined",
		"type": "undefined",
		"userId": "undefined",
		"appId": "test-service",
		"clusterId": "undefined"
	},
	"content": "<Buffer 6e 75 6c 6c>"
}
```

## TSL/SSL support
To configure certificates and learn why do you need it, [read here](https://www.rabbitmq.com/ssl.html).

To use `amqps` connection:

``` typescript
RMQModule.forRoot({
	exchangeName: 'test',
	connections: [
		{
			protocol: RMQ_PROTOCOL.AMQPS, // new
			login: 'admin',
			password: 'admin',
			host: 'localhost',
		},
	],
	connectionOptions: {
		cert: fs.readFileSync('clientcert.pem'),
		key: fs.readFileSync('clientkey.pem'),
		passphrase: 'MySecretPassword',
		ca: [fs.readFileSync('cacert.pem')]
	} // new
}),
```

This is the basic example with reading files, but you can do however you want. `cert`, `key` and `ca` must be Buffers. Notice: `ca` is array. If you don't need keys, just use `RMQ_PROTOCOL.AMQPS` protocol.

To use it with `pkcs12` files:

``` typescript
connectionOptions: {
	pfx: fs.readFileSync('clientcertkey.p12'),
	passphrase: 'MySecretPassword',
	ca: [fs.readFileSync('cacert.pem')]
},
```

## Manual message Ack/Nack

If you want to use your own [ack](https://www.squaremobius.net/amqp.node/channel_api.html#channel_nack)/[nack](https://www.squaremobius.net/amqp.node/channel_api.html#channel_ack) logic, you can set manual acknowledgement to `@RMQRoute`. Than in any place you have to manually ack/nack message that you get with `@RMQMessage`.

```typescript
import { RMQRoute, Validate, RMQMessage, ExtendedMessage, RMQService } from 'nestjs-rmq';

@Controller()
export class MyController {
	constructor(private readonly rmqService: RMQService) {}

	@RMQRoute('my.rpc', { manualAck: true })
	myMethod(data: myClass, @RMQMessage msg: ExtendedMessage): number {
		// Any logic goes here
		this.rmqService.ack(msg);
		// Any logic goes here
	}

	@RMQRoute('my.other-rpc', { manualAck: true })
	myOtherMethod(data: myClass, @RMQMessage msg: ExtendedMessage): number {
		// Any logic goes here
		this.rmqService.nack(msg);
		// Any logic goes here
	}
}
```

## Send debug information to error or log

`ExtendedMessage` has additional method to get all data from message to debug it. Also it serializes content and hides Buffers, because they can be massive. Then you can put all your debug info into Error or log it.

```typescript
import { RMQRoute, Validate, RMQMessage, ExtendedMessage, RMQService } from 'nestjs-rmq';

@Controller()
export class MyController {
	constructor(private readonly rmqService: RMQService) {}

	@RMQRoute('my.rpc')
	myMethod(data: myClass, @RMQMessage msg: ExtendedMessage): number {
		// ...
		console.log(msg.getDebugString());
		// ...
	}
}
```

You will get info about message, field and properties:

```json
{
	"fields": {
		"consumerTag": "amq.ctag-Q-l8A4Oh76cUkIKbHWNZzA",
		"deliveryTag": 4,
		"redelivered": false,
		"exchange": "test",
		"routingKey": "debug.rpc"
	},
	"properties": {
		"headers": {},
		"correlationId": "388236ad-6f01-3de5-975d-f9665b73de33",
		"replyTo": "amq.rabbitmq.reply-to.g1hkABNyYWJiaXRANzQwNDVlYWQ5ZTgwAAAG2AAAAABfmnkW.9X12ySrcM6BOXpGXKkR+Yg==",
		"timestamp": 1603959908996,
		"appId": "test-service"
	},
	"message": {
		"prop1": [1],
		"prop2": "Buffer - length 11"
	}
}
```

## Customizing massage with msgFactory

`@RMQRoute` handlers accepts a single parameter `msg` which is a ampq `message.content` parsed as a JSON. You may want to add additional custom layer to that message and change the way handler is called. For example, you may want to structure your message with two different parts: payload (containing actual data) and appId (containing request applicationId) and process them explicitly in your handler.

To do that, you may pass a param to the `RMQRoute` a custom message factory `msgFactory?: (msg: Message) => any;`.

The default msgFactory:

```typescript
@RMQRoute('topic', {
	msgFactory: (msg: Message) => JSON.parse(msg.content.toString())
})
```

Custom msgFactory that returns additional argument (sender appId) and change request:

```typescript
@RMQRoute(CustomMessageFactoryContracts.topic, {
	msgFactory: (msg: Message) => {
		const content: CustomMessageFactoryContracts.Request = JSON.parse(msg.content.toString());
		content.num = content.num * 2;
		return [content, msg.properties.appId];
	}
})
customMessageFactory({ num }: CustomMessageFactoryContracts.Request, appId: string): CustomMessageFactoryContracts.Response {
	return { num, appId };
}
```

## Validating data

NestJS-rmq uses [class-validator](https://github.com/typestack/class-validator) to validate incoming data. To use it, decorate your route method with `Validate`:

```typescript
import { RMQRoute, Validate } from 'nestjs-rmq';

@RMQRoute('my.rpc')
@Validate()
myMethod(data: myClass): number {
	// ...
}
```

Add it after `@RMQRoute()`. Where `myClass` is data class with validation decorators:

```typescript
import { IsString, MinLength, IsNumber } from 'class-validator';

export class myClass {
	@MinLength(2)
	@IsString()
	name: string;

	@IsNumber()
	age: string;
}
```

If your input data will be invalid, the library will send back an error without even entering your method. This will prevent you from manually validating your data inside route. You can check all available validators [here](https://github.com/typestack/class-validator).

## Using pipes

To intercept any message to any route, you can use `@RMQPipe` decorator:

```typescript
import { RMQRoute, RMQPipe } from 'nestjs-rmq';

@RMQPipe(MyPipeClass)
@RMQRoute('my.rpc')
myMethod(numbers: number[]): number {
	//...
}
```

where `MyPipeClass` extends `RMQPipeClass` with one method `transform`:

```typescript
class MyPipeClass extends RMQPipeClass {
	async transfrom(msg: Message): Promise<Message> {
		// do something
		return msg;
	}
}
```

## Using RMQErrorHandler

If you want to use custom error handler for dealing with errors from replies, use `errorHandler` in module options and pass class that extends `RMQErrorHandler`:

```typescript
class MyErrorHandler extends RMQErrorHandler {
	public static handle(headers: IRmqErrorHeaders): Error | RMQError {
		// do something
		return new RMQError(
			headers['-x-error'],
			headers['-x-type'],
			headers['-x-status-code'],
			headers['-x-data'],
			headers['-x-service'],
			headers['-x-host']
		);
	}
}
```

## HealthCheck

RQMService provides additional method to check if you are still connected to RMQ. Although reconnection is automatic, you can provide wrong credentials and reconnection will not help. So to check connection for Docker healthCheck use:

```typescript
const isConnected = this.rmqService.healthCheck();
```

If `isConnected` equals `true`, you are successfully connected.

## Disconnecting

If you want to close connection, for example, if you are using RMQ in testing tools, use `disconnect()` method;

## Unit and E2e tests

### Using in tests

RMQ library supports using RMQ module in your test suites without needing RabbitMQ instance. To use library in tests, use `forTest` method in module.

```typescript
	import { RMQTestService } from 'nestjs-rmq';

	let rmqService: RMQTestService;

	beforeAll(async () => {
		const apiModule = await Test.createTestingModule({
			imports: [
				RMQModule.forTest({})
			],
			controllers: [MicroserviceController],
		}).compile();
		api = apiModule.createNestApplication();
		await api.init();

		rmqService = apiModule.get(RMQService);
	});
```

You can pass any options you pass in normal `forRoot` (except `errorHandler`).

From module, you will get `rmqService` which is similar to normal service, with two additional methods:

- `triggerRoute` - trigger your RMQRoute, simulating incoming message.
- `mockReply` - mock reply if you are using `send` method.
- `mockError` - mock error if you are using `send` method.

### triggerRoute

Emulates message received buy your RMQRoute.

```typescript
const { result } = await rmqService.triggerRoute<Request, Response>(topic, data);
```

- `topic` - topic, that you want to trigger (pattern supported).
- `data` - data to send in your method.

### mockReply

If your service needs to send data to other microservice, you can emulate its reply with:

```typescript
rmqService.mockReply(topic, res);
```

- `topic` - all messages sent to this topic will be mocked.
- `res` - mocked response data.

After this, all `rmqService.send(topic, { ... })` calls will return `res` data.

### mockError

If your service needs to send data to other microservice, you can emulate its error with:

```typescript
rmqService.mockError(topic, error);
```

- `topic` - all messages sent to this topic will be mocked.
- `error` - error that `send` method will throw.

After this, all `rmqService.send(topic, { ... })` calls will throw `error`.

## Contributing

For e2e tests you need to install Docker in your machine and start RabbitMQ docker image with `docker-compose.yml` in `e2e` folder:

```
docker-compose up -d
```

Then change IP in tests to `localhost` and run tests with:

```
npm run test:e2e
```

![alt cover](https://github.com/AlariCode/nestjs-rmq/raw/master/img/tests.png)


For unit tests just run:

```
npm run test
```


### Migrating from version 1

New version of nestjs-rmq contains minor breaking changes, and is simple to migrate to.

-   `@RMQController` decorator is deprecated.
    You will get warning if you continue to use it, and it will be deleted in future versions.
    You can safely remove it from a controller or service. `msgFactory` inside options will not be functional anymore. You have to move it to `@RMQRoute`
-   `msgFactory` changed its interface from

```typescript
msgFactory?: (msg: Message, topic: IRouteMeta) => any[];
```

to

```typescript
msgFactory?: (msg: Message) => any[];
```

because all `IRouteMeta` already contained in `Message`.

-   `msgFactory` can be passed to `@RMQRoute` instead of `@RMQController`
