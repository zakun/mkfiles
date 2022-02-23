## RabbitMQ-工作队列


![1](https://www.rabbitmq.com/img/tutorials/python-two.png)

### Message acknowledgmen

Message acknowledgments were previously turned off by ourselves. It's time to turn them on by setting the fourth parameter to `basic_consume` to `false` (true means no ack) and send a proper acknowledgment from the worker, once we're done with a task.

```
$callback = function ($msg) {
  echo ' [x] Received ', $msg->body, "\n";
  sleep(substr_count($msg->body, '.'));
  echo " [x] Done\n";
  $msg->ack();
};

$channel->basic_consume('task_queue', '', false, false, false, false, $callback);
```

### Forgotten acknowledgment

It's a common mistake to miss the `ack`. It's an easy error, but the consequences are serious. Messages will be redelivered when your client quits (which may look like random redelivery), but RabbitMQ will eat more and more memory as it won't be able to release any unacked messages.

In order to debug this kind of mistake you can use `rabbitmqctl` to print the `messages_unacknowledged` field:

```
sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged
```

On Windows, drop the sudo:

```
rabbitmqctl.bat list_queues name messages_ready messages_unacknowledged
```

### Message durability

First, we need to make sure that the queue will survive a RabbitMQ node restart. In order to do so, we need to declare it as durable. To do so we pass the third parameter to queue_declare as true:

```
$channel->queue_declare('hello', false, true, false, false);
```

Although this command is correct by itself, it won't work in our present setup. That's because we've already defined a queue called `hello` which is not durable. RabbitMQ doesn't allow you to redefine an existing queue with different parameters and will return an error to any program that tries to do that. But there is a quick workaround - let's declare a queue with different name, for example `task_queue`:

```
$channel->queue_declare('task_queue', false, true, false, false);
```

This flag set to `true` needs to be applied to both the producer and consumer code.

At this point we're sure that the `task_queue` queue won't be lost even if RabbitMQ restarts. Now we need to mark our messages as persistent - by setting the `delivery_mode = 2` message property which `AMQPMessage` takes as part of the property array.

```
$msg = new AMQPMessage(
    $data,
    array('delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT)
);

```

### Fair dispatch

![2](https://www.rabbitmq.com/img/tutorials/prefetch-count.png)

In order to defeat that we can use the `basic_qos` method with the `prefetch_count = 1` setting. This tells RabbitMQ not to give more than one message to a worker at a time. Or, in other words, don't dispatch a new message to a worker until it has processed and acknowledged the previous one. Instead, it will dispatch it to the next worker that is not still busy.

```
$channel->basic_qos(null, 1, null);
```

### Putting it all together

Final code of our `new_task.php` file:

```
<?php

require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();

$channel->queue_declare('task_queue', false, true, false, false);

$data = implode(' ', array_slice($argv, 1));
if (empty($data)) {
    $data = "Hello World!";
}
$msg = new AMQPMessage(
    $data,
    array('delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT)
);

$channel->basic_publish($msg, '', 'task_queue');

echo ' [x] Sent ', $data, "\n";

$channel->close();
$connection->close();
```

And our `worker.php`:

```
<?php

require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;

$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();

$channel->queue_declare('task_queue', false, true, false, false);

echo " [*] Waiting for messages. To exit press CTRL+C\n";

$callback = function ($msg) {
    echo ' [x] Received ', $msg->body, "\n";
    sleep(substr_count($msg->body, '.'));
    echo " [x] Done\n";
    $msg->ack();
};

$channel->basic_qos(null, 1, null);
$channel->basic_consume('task_queue', '', false, false, false, false, $callback);

while ($channel->is_open()) {
    $channel->wait();
}

$channel->close();
$connection->close();
```