Exrabbit
========

Elixir client for RabbitMQ, based on [rabbitmq-erlang-client][1].

  [1]: https://github.com/rabbitmq/rabbitmq-erlang-client


## Rationale

This project's aim is not to be a complete replacement for the Erlang client.
Instead, it captures common usage patterns and exposes them as a higher level
API.

Climbing the ladder of abstraction even higher, Exrabbit provides a set of DSLs
that make writing common types of AMQP producers and consumers a breeze.


## Installation

Add Exrabbit as a dependency to your project:

```elixir
def application do
  [applications: [:exrabbit]]
end

def deps do
  [{:exrabbit, github: "inbetgames/exrabbit"}]
end
```


## Configuration

Exrabbit can be configured using `config.exs` (see the bundled one for the
available settings) or YAML (via [Sweetconfig][2]).

  [2]: https://github.com/inbetgames/sweetconfig

Example `config.exs`:

```elixir
use Mix.Config

config :exrabbit,
  host: "localhost",
  username: "guest",
  password: "guest",
  confirm_timeout: 5000
```


## Usage (DSL)


## Usage (basic)

In all examples below we will assume the following aliases have been defined:

```elixir
alias Exrabbit.Connection, as: Conn
alias Exrabbit.Producer
alias Exrabbit.Consumer

# this call is needed when working with records
require Exrabbit.Records
```

### Aside: working with records

In order to provide the complete functionality implemented by the Erlang
client, in some cases Exrabbit relies on Erlang records that represent AMQP
methods. A single method is an instance of a record and it is executed on a
channel.

See `doc/records.md` for an overview of which records have been inherited from
the Erlang client and which ones have been superceded by a higher level API.

### Publishing to a queue

A basic example of a publisher:

```elixir
# Open both a connection to the broker and a channel in one go
conn = %Conn{channel: chan} = Conn.open(with_channel: true)

# Create a producer on the chan
# producer is just a struct encapsulating the exchange (default one in this case)
# and the queue
producer = Producer.new(chan, queue: "hello_queue")
Producer.publish(producer, "message")
Producer.publish(producer, "bye-bye")

# Close the channel and connection in one go
Conn.close(conn)
```

One can also feed a stream of binaries into a producer:

```elixir
# Create a producer on the chan
stream = IO.binstream(:stdio, :line)
producer = Producer.new(chan, queue: "hello_queue")
Enum.into(producer, stream)
```

To adjust properties of a queue, one can use a record for the queue:

```elixir
conn = %Conn{channel: chan} = Conn.open(with_channel: true)

# we are using a record provided by the Erlang client here
queue = Exrabbit.Records.queue_declare(queue: "name", auto_delete: true, exclusive: false)
producer = Producer.new(chan, queue: queue)
Producer.publish(producer, "message")
```

### Publishing to an exchange

Most of the time you'll be working with exchanges because they provide a more
flexible way to route messages to different queues and eventually consumers.

```elixir
# :with_channel is true by default
conn = %Conn{channel: chan} = Conn.open()

# again, using a record from the Erlang client
exchange = Exrabbit.Records.exchange_declare(exchange: "logs", type: "fanout")
fanout = Producer.new(chan, exchange: exchange)

Producer.publish(fanout, "[info] some log")
Producer.publish(fanout, "[error] crashed")


exchange = Exrabbit.Records.exchange_declare(exchange: "more_logs", type: "topic")
topical = Producer.new(chan, exchange: exchange)

Producer.publish(topical, "some log", routing_key: "logs.info")
Producer.publish(topical, "crashed", routing_key: "logs.error")

Conn.close(conn)
```

### Receiving messages

When receiving messages, the client sets up a queue, binds it to an exchange
and subscribes to the queue to be notified of incoming messages:

```elixir
conn = %Conn{channel: chan} = Conn.open()

topical_exchange = Exrabbit.Records.exchange_declare(exchange: "more_logs", type: "topic")

subscription_fun = fn
  {:begin, _tag} -> IO.puts "Did subscribe"
  {:end, _tag} -> IO.puts "Subscription ended"
  {:msg, _tag, message} -> IO.puts "Received message from queue: #{message}"
end

# bind the queue to the exchange and subscribe to it
consumer =
  Consumer.new(chan, exchange: topical_exchange, new_queue: "")
  |> Consumer.subscribe(subscription_fun)

# arriving messages will be consumed by subscription_fun
# ...

Consumer.unsubscribe(consumer)

Conn.close(conn)
```

There is also a way to request messages one by one using the `get` function:

```elixir
# assume we have set up the channel as before

consumer = Consumer.new(chan, exchange: topical_exchange, queue: queue)
{:ok, message} = Consumer.get(consumer)
nil = Consumer.get(consumer)
```

### Producer confirms and transactions

An open channel can be switched to confirm-mode or tx-mode.

In confirm mode each published message will be ack'ed or nack'ed by the broker.

In tx-mode one has to call `Exrabbit.Producer.commit` after sending a batch of
messages. Those messages will be delivered atomically: either all or nothing.

See `doc/producer_basic.md` for examples.

### Consumer acknowledgements

When receiving messages, consumers may specify whether the broker should wait
for acknowledgement before removing a message from the queue.

See `doc/consumer_basic.md` for examples.


## OLD README ##

Easy way to get a queue/exchange worker:


```elixir
import Exrabbit.DSL

amqp_worker TestQ, queue: "testQ" do
  on json = %{} do
    IO.puts "JSON: #{inspect json}"
  end
  on <<"hello">> do
    IO.puts "Hello-hello from MQ"
  end
  on text do
    IO.puts "Some random binary: #{inspect text}"
  end
end
```

N.B. Instead of passing configuration options when defining module with `amqp_worker` one can add following to config.exs:

```elixir
[
  exrabbit: [
    my_queue: [queue: "TestQ"]
  ]
]
```

and then define module as:


```elixir
amqp_worker TestQ, config_name: :my_queue, decode_json: [keys: :atoms] do
  on %{cmd: "resize_image", image: image} do
    IO.puts "Resizing image: #{inspect image}"
  end
end
```


Checking if message was published:


```elixir
publish(channel, exchange, routing_key, message, :wait_confirmation)
```


Workflow to send message:


```elixir
amqp = Exrabbit.Utils.connect
channel = Exrabbit.Utils.channel amqp
Exrabbit.Utils.publish channel, "testExchange", "", "hello, world"
```


To get messages, almost the same, but functions are


```elixir
Exrabbit.Utils.get_messages channel, "testQueue"
case Exrabbit.Utils.get_messages_ack channel, "testQueue" do
	nil -> IO.puts "No messages waiting"
	[tag: tag, content: message] ->
		IO.puts "Got message #{message}"
		Exrabbit.Utils.ack tag # acking message
end
```


Please consult: http://www.rabbitmq.com/erlang-client-user-guide.html#returns to find out how to write gen_server consuming messages.


```elixir
defmodule Consumer do
  use GenServer.Behaviour
  import Exrabbit.Utils
  require Lager

  def start_link, do: :gen_server.start_link(Consumer, [], [])

  def init(_opts) do
    amqp = connect
    channel = channel amqp
    subscribe channel, "testQ"
    {:ok, [connection: amqp, channel: channel]}
  end

  def handle_info(request, state) do
    case parse_message(request) do
      nil -> Lager.info "Got nil message"
      {tag, payload} ->
        Lager.info "Got message with tag #{tag} and payload #{payload}"
        ack state[:channel], tag
    end
    { :noreply, state}
  end
end
```


Or same, using behaviours:


```elixir
defmodule Test do
  use Exrabbit.Subscriber

  def handle_message(msg, _state) do
    case parse_message(msg) do
      nil ->
        IO.puts "Nil"
      {tag,json} ->
        IO.puts "Msg: #{json}"
        ack _state[:channel], tag
      {tag,json,reply_to} ->
        IO.puts "For RPC messaging: #{json}"
        publish(_state[:channel], "", reply_to, "#{json}") # Return ECHO
        ack _state[:channel], tag
    end
  end
end

:gen_server.start Test, [queue: "testQ"], []
```




