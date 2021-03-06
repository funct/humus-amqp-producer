# Message Producers

There are four types of message producers shipped with the library.
The command producer, delayed command producer, event producer and query producer.

First, an example configuration (the configuration of connection, exchanges and queues is skipped):

```php
return [
    'humus' => [
        'amqp' => [
            'producer' => [
                'command_producer' => [
                    'type' => 'json',
                    'exchange' => 'command_bus_exchange',
                ],
                'delayed_command_producer' => [
                    'type' => 'json',
                    'exchange' => 'delayed_command_bus_exchange',
                ],
                'event_producer' => [
                    'type' => 'json',
                    'exchange' => 'event_bus_exchange',
                ],
                'query_producer' => [
                    'type' => 'json',
                    'exchange' => 'query_bus_exchange',
                ],
            ],
        ],
    ],
    'prooph' => [
        'humus-amqp-producer' => [
            'message_producer' => [
                'amqp_command_producer' => [
                    'producer'          => 'command_producer',
                    'app_id'            => 'my app',
                    'message_converter' => \Prooph\Common\Messaging\NoOpMessageConverter::class,
                ],
                'amqp_event_producer' => [
                    'producer'          => 'event_producer',
                    'app_id'            => 'my app',
                    'message_converter' => \Prooph\Common\Messaging\NoOpMessageConverter::class,
                ],
                'amqp_query_producer' => [
                    'producer'          => 'query_producer',
                    'app_id'            => 'my app',
                    'message_converter' => \Prooph\Common\Messaging\NoOpMessageConverter::class,
                ],
            ],
            'delayed_message_producer' => [
                'amqp_delayed_command_producer' => [
                    'producer'          => 'delayed_command_producer',
                    'app_id'            => 'my app',
                    'message_converter' => \Prooph\Common\Messaging\NoOpMessageConverter::class,
                ],
            ],
        ],
    ],
    'dependencies' => [
        'factories' => [
            'command_producer' => [
                \Humus\Amqp\Container\ProducerFactory::class,
                'command_producer'
            ],
            'delayed_command_producer' => [
                \Humus\Amqp\Container\ProducerFactory::class,
                'delayed_command_producer'
            ],
            'event_producer' => [
                \Humus\Amqp\Container\ProducerFactory::class,
                'event_producer'
            ],
            'query_producer' => [
                \Humus\Amqp\Container\ProducerFactory::class,
                'query_producer'
            ],
            'amqp_command_producer' => [
                \Prooph\ServiceBus\Message\HumusAmqp\Container\AmqpMessageProducerFactory::class,
                'amqp_command_producer'
            ],
            'amqp_delayed_command_producer' => [
                \Prooph\ServiceBus\Message\HumusAmqp\Container\AmqpDelayedMessageProducerFactory::class,
                'amqp_delayed_command_producer'
            ],
            'amqp_event_producer' => [
                \Prooph\ServiceBus\Message\HumusAmqp\Container\AmqpMessageProducerFactory::class,
                'amqp_event_producer'
            ],
            'amqp_query_producer' => [
                \Prooph\ServiceBus\Message\HumusAmqp\Container\AmqpMessageProducerFactory::class,
                'amqp_query_producer'
            ],
        ],
    ],
];
```

In the `humus` section the bare metal producers are configured.
The the `prooph` - `humus-amqp-producer` the producers are wrapped with some additional stuff needed.
The most important is the used `message_converter`. The app_id is an id that is passed to AMQP, so you can see
from which application a message was coming.

You can then use the message producer directly:

```php
$producer = $container->get('amqp_command_producer');
$producer($command);

$producer = $container->get('amqp_query_producer');
$producer($query, $deferred);
```

You can also route a specific message to the required producer, f.e. an command bus here:

```php
return [
    'prooph' => [
        'service_bus' => [
            'command_bus' => [
                'router' => [
                    'routes' => [
                        \Prooph\Snapshotter\TakeSnapshot::class => 'amqp_command_producer',
                    ],
                ],
            ],
        ],
    ],
];
```

You can also route all messages to the required producer, f.e. an event bus here:

```php
return [
    'prooph' => [
        'service_bus' => [
            'event_bus' => [
                'plugins' => [
                    'amqp_event_producer_plugin',
                ],
            ],
        ],
    ],
    'dependencies' => [
        'factories' => [
            'amqp_event_producer_plugin' => [
                \Prooph\ServiceBus\Message\HumusAmqp\Container\AmqpMessageProducerFactory,
                'amqp_event_producer'
            ],
        ],
    ],
];
```

# Delayed Messages

Delayed messages are messages that should be executed at a specific point in time.
F.e. execute a command in 5min from now on.

In order to make it work, you need to configure a delayed message exchange and configure it accordingly.
The RabbitMQ extension [delayed-message-exchange](https://github.com/rabbitmq/rabbitmq-delayed-message-exchange) is required.

A delayed message must implement `Prooph\ServiceBus\Message\HumusAmqp\DelayedMessage`.
If you are using `Prooph\Common\Messaging\Command` as your base command class, you can simply use
`Prooph\ServiceBus\Message\HumusAmqp\DelayedCommand` instead.

## Usage

```php
$command = SomeCommand::withData(['foo' => 'bar'])->executeAt((new DateTimeImmutable('now'))->modify('+5 mins'));

$commandBus->dispatch($command);
```

# Query producer

With an amqp query producer you can do two things:

a) send a query to a remote server and get a response
b) send a parallel query to multiple remote servers and thus allow quering in parallel

The latter is especially useful if you need to do multiple larger queries in a webrequest, so those can be resolved
in a shorter period of time.

## Usage

For configuration example, see above.

```php
$promise = $queryBus->dispatch($query);
$promise->then(function (array $res): void {
    // do something
});

$promise->otherwise(function () {
    // do something else
});
```

## Parallel messages

A parallel message has to implement `Prooph\ServiceBus\Message\HumusAmqp\ParallelMessage`. This is usually helpful
for queries to be executed parallel.
 
### Example code

```php
<?php

declare(strict_types=1);

namespace My\App;

use Humus\Amqp\JsonRpc\ResponseCollection;
use Prooph\Common\Messaging\Message;
use Prooph\Common\Messaging\Query;
use Prooph\ServiceBus\Message\HumusAmqp\ParallelMessage;

class MyParallelQuery extends Query implements ParallelMessage
{
    private $messages = [];
    
    public static function withQueries(Message ...$queries): MyParallelQuery
    {
        $self = new self();
        $self->messages = $queries;
        
        return $self;
    }
    
    protected function setPayload(array $payload): void
    {
    }
    
    public function messages(): array
    {
        return $this->messages;
    }
}

$query1 = new SomeQuery(['foo' => 'bar']);
$query2 = new SomeOtherQuery(['foo' => 'baz']);

$query = MyParallelQuery::withQueries($query1, $query2);

$promise = $this->eventBus->dispatch($query);
$promise->then(function (ResponseCollection $collection): void {
    // do something
});

$promise->otherwise(function () {
    // do something else
});
```

The result of a parallel query will always return a response collection object.
