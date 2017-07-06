# RedisSMQ - Yet another simple Redis message queue

A simple high-performance Redis message queue for Node.js.

## Features

 * Persistent: No messages are lost in case of a consumer failure.
 * Atomic: A message is delivered only once to one consumer (in FIFO order) so you would never fall into a situation
 where a message could be processed more than once.
 * Fast: 14k+ messages/second on a virtual machine of 4 CPU cores and 8GB RAM and running one consumer.
 * Concurrent consumers: A queue can be consumed by many consumers, launched on the same or on different hosts.
 * Message TTL: A message will expire and not be consumed if it has been in the queue for longer than the TTL.
 * Message consume timeout: The amount of time for a consumer to consume a message. If the timeout exceeds,
 message processing is cancelled and the message is re-queued to be consumed again.
 * Highly optimized: No promises, no async/await, small memory footprint, no memory leaks.
 * Monitorable: Statistics (input/processing/acks/unacks messages rates, online consumers, queues, etc.)
   are provided in real-time.
 * Logging: Supports JSON log format for troubleshooting and analytics purposes.
 * Configurable: Many options and features can be configured.
 
## Installation

```text
npm install redis-smq --save
```

Considerations:

This library make use of many of ES6 features including:

- arrow functions
- default function parameters
- destructing assignment
- template literals
- const, let, block-level function declaration
- symbols
- classes

Minimal Node.js version support is 6.5. The latest stable Node.js version is recommended. 

## Configuration 

RedisSMQ configuration parameters:

- `redis.host` *(String): Required.* IP address of the Redis server.
- `redis.port` *(Integer): Required.* Port of the Redis server.
- `log` *(Object): Optional.* Logging parameters.
- `log.enabled` *(Integer/Boolean): Optional.* Enable/disable logging. By default logging is disabled.
- `log.options` *(Object): Optional.* All valid Bunyan configuration options are accepted. Please look at the 
  [Bunyan Repository](https://github.com/trentm/node-bunyan) for more details.
- `monitor` *(Object): Optional.* RedisSMQ monitor parameters.
- `monitor.enabled` *(Boolean/Integer): Optional.* Enable/Disable the monitor.
- `monitor.host` *(String): Optional.* IP address of the monitor server.
- `monitor.port` *(Integer): Optional.* Port of the monitor server.

### Configuration example

```javascript
// filename: ./example/config/index.js

'use strict';

const path = require('path');

module.exports = {
    redis: {
        host: '127.0.0.1',
        port: 6379,
    },
    log: {
        enabled: 0,
        options: {
            level: 'trace',
            /*
            streams: [
                {
                    path: path.normalize(`${__dirname}/../logs/redis-smq.log`)
                },
            ],
            */
        },
    },
    monitor: {
        enabled: true,
        host: '127.0.0.1',
        port: 3000,
    },
};
```

## Usage

### Definitions

**Message TTL** - All queue messages are guaranteed to not be consumed and destroyed if they have been in the queue for 
longer than an amount of time called TTL (time-to-live) in milliseconds. Message TTL can be set per message when 
producing it or per consumer (for all queue messages).

**Message consumption timeout** - Also called job timeout, is the amount of time in milliseconds before a consumer 
consuming a message times out. If the consumer does not consume the message within the set time limit, the message 
consumption is automatically canceled and the message is re-queued to be consumed again.

**Message retry threshold** - Failed messages are re-queued and consumed again unless retry threshold exceeded. If the 
retry threshold exceeds, the messages are moved to a dead-letter queue (DLQ).

**Dead-Letter Queue** - Each queue has a system generated corresponding dead-letter queue where all failed to consume 
messages are moved to. Messages from a dead-letter queue can only read and deleted (through the monitor). 

**Message acknowledgment** - The message acknowledgment informs the consumer that the message has been consumed 
successfully. If an error occurred during the processing of a message, the error should be sent back to the consumer 
so the message is re-queued again and considered to be unacknowledged.

### Producer

* Producer class constructor `Producer(queueName, config)`:
  * `queueName` *(string): Required.* The name of the queue where produced messages are queued.
  * `config` *(object): Required.* Configuration object.
* A Producer instance provides 2 methods:
  * `produce(message, cb)`: Produce a message.
    * `message` *(mixed): Required.* The actual message, to be consumed by a consumer.
    * `cb(err)` *(function): Required.* Callback function. When called without error argument, the message is 
    successfully published.
  * `produceWithTTL(message, ttl, cb)`: Produce a message with TTL (time-to-live).
    * `message` *(mixed): Required.* The actual message, to be consumed by a consumer.
    * `ttl` *(Integer): Required.* Message TTL in milliseconds.
    * `cb(err)` *(function): Required.* Callback function. When called without error argument, the message is 
    successfully published.

#### Producer example

```javascript
// filename: ./example/test-queue-producer-launch.js

const config = require('./config');
const Producer = require('redis-smq').Producer;
const producer = new Producer('test_queue', config);

producer.produce({'hello': 'world'}, (err) => {
    if (err) {
        console.error(err);
        process.exit(1);
    }
    console.log('Successfully published!');
});

producer.produceWithTTL({'hello': 'world'}, 60000, (err) => {
    if (err) {
        console.error(err);
        process.exit(1);
    }
    console.log('Successfully published!');
});
```

### Consumer

* Consumer class constructor `Consumer(config, options)`:
  * `config` *(object): Required.* Configuration object.
  * `options` *(object): Optional.* Consumer configuration parameters.
  * `options.messageConsumeTimeout` *(Integer): Optional.* Consumer timeout for consuming a message, in milliseconds. 
  By default message consumption timeout is not set. 
  * `options.messageTTL` *(Integer): Optional.* Message TTL in milliseconds. By default messageTTL is not set.
  * `options.messageRetryThreshold` *(Integer): Optional.* Message retry threshold. By default message retry threshold 
  is set to 3.
* Consumers classes are saved per files. Each consumer file represents a consumer class.
* Each consumer class:
  * Extends redisSMQ.Consumer class.
  * Has a static property 'queueName' - The name of the queue to consume messages from.
  * Required to have a `consume(message, cb)` method which is called each time a message received:
    * `message` *(mixed): Required.* Actual message payload published by a producer
    * `cb(err)` *(function): Required.* Callback function. When called with an error argument the message is 
    unacknowledged. Otherwise (if called without arguments) the message is acknowledged.

#### Consumer example

```javascript
// filename: ./example/consumers/test-queue-consumer.js

const redisSMQ = require('redis-smq');
const Consumer = redisSMQ.Consumer;

class TestQueueConsumer extends Consumer {

    consume(message, cb) {
        //console.log(`Got message to consume: `, JSON.stringify(message));

        //throw new Error('TEST!');

        //cb(new Error('TEST!'));

        //const timeout = parseInt(Math.random() * 100);
        //setTimeout(cb, timeout);

        cb();
    }
}

TestQueueConsumer.queueName = 'test_queue';

module.exports = TestQueueConsumer;
```

#### Running a consumer

Launch file:

```javascript
// filename: ./example/test-queue-consumer-launch.js

'use strict';

const config = require('./config');
const TestQueueConsumer = require('./consumers/test-queue-consumer');

const consumer = new TestQueueConsumer(config, { consumeTimeout: 2000 });
consumer.run();
```

Running from CLI:

```text
$ node test-queue-consumer-launch.js
```
## Troubleshooting and monitoring

### Logs

This package is using JSON log format, thanks to [Bunyan](https://github.com/trentm/node-bunyan).

The structured data format of JSON allows analytics tools to take place but also helps to monitor and troubleshoot 
issues easier and faster.

By default all logs are disabled. Logging can affect performance (due to I/O operations). When enabled you can 
use bunyan utility to pretty format the output.

Unless configured otherwise, the standard output is the console which launched the consumer.

```text
$ node consumer | ./node_modules/.bin/bunyan
```
### Monitoring

The RedisSMQ Monitoring is an interface which let you monitor and debug your RedisSMQ server from a web browser in 
real-time.

First enable monitoring in your configuration file and provide monitoring server parameters.

Monitor server example:

```javascript
// filename: ./example/monitor.js

'use strict';

const config = require('./config');
const monitorServer = require('redis-smq').monitor(config);

monitorServer.listen(() => {
    console.log('Monitor server is running...');
});

```

Launching the server:

```text
$ node monitor.js
```
#### RedisSMQ Monitor screenshots

Please note that the numbers shown in the screenshots are related with Redis server configuration and host performance 
parameters on which the server is running!

##### Redis-SMQ running 30 consumers (15 consumers per 2 queues) messages while other producers are producing messages in parallel:

![RedisSMQ Monitor](./screenshots/img_1.png)

More screenshots, could be found in the [screenshots folder](https://github.com/weyoss/redis-smq/tree/master/screenshots)

## Bugs

If you find any bugs, please let me know. Open a [issue](https://github.com/weyoss/redis-smq/issues) into github 
(including the case to reproduce the bug when possible).

## License

[MIT](https://github.com/weyoss/redis-smq/blob/master/LICENSE)