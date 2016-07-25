# cpp_redis
cpp_redis is C++11 Asynchronous Redis Client.

Network is based on raw sockets API, making the library really lightweight.
Commands pipelining is supported.

## Requirements
* C++11
* Mac or Linux (no support for Windows platforms)

## Compiling
The library uses `cmake`. In order to build the library, follow these steps:

```bash
git clone https://github.com/Cylix/cpp_redis.git
cd cpp_redis
./install_deps.sh # necessary only for building tests
mkdir build
cd build
cmake .. # only library
cmake .. -DBUILD_TESTS=true # library and tests
cmake .. -DBUILD_EXAMPLES=true # library and examples
cmake .. -DBUILD_TESTS=true -DBUILD_EXAMPLES=true # library, tests and examples
make -j
```

If you want to install the library in a specific folder:

```bash
cmake -DCMAKE_INSTALL_PREFIX=/destination/path ..
make install -j
```

Then, you just have to include `<cpp_redis/cpp_redis>` in your source files and link the `cpp_redis` library with your project.

To build the tests, it is necessary to install `google_tests`. Just run the `install_deps.sh` script which does the work for you.

## Redis Client
`redis_client` is the class providing communication with a redis server.

### Methods

#### void connect(const std::string& host = "127.0.0.1", unsigned int port = 6379, const disconnection_handler& handler = nullptr)
Connect to the Redis Server. Connection is done synchronously.
Throws redis_error in case of failure or if client if already connected.

Also set the disconnection handler which is called whenever a disconnection has occurred.
Disconnection handler is an `std::function<void(redis_client&)>`.

#### void disconnect(void)
Disconnect client from remote host.
Throws redis_error if client is not connected to any server.

#### bool is_connected(void)
Returns whether the client is connected or not.

#### redis_client& send(const std::vector<std::string>& redis_cmd, const reply_callback& callback = nullptr)
Send a command and set the callback which has to be called when the reply has been received.
If `nullptr` is passed as callback, command is executed and no callback will be called.
Reply callback is an `std::function<void(reply&)>`.

The command is not effectively sent immediately, but stored inside an internal buffer until `commit()` is called.

### redis_client& commit(void)
Send all the commands that have been stored by calling `send()` since the last `commit()` call to the redis server.

That is, pipelining is supported in a very simple and efficient way: `client.send(...).send(...).send(...).commit()` will send the 3 commands at once (instead of sending 3 network requests, one for each command, as it would have been done without pipelining).


### Example

```cpp
#include <cpp_redis/cpp_redis>

#include <signal.h>
#include <iostream>

volatile std::atomic_bool should_exit(false);
cpp_redis::redis_client client;

void
sigint_handler(int) {
  std::cout << "disconnected (sigint handler)" << std::endl;
  client.disconnect();
  should_exit = true;
}

int
main(void) {
  client.connect("127.0.0.1", 6379, [] (cpp_redis::redis_client&) {
    std::cout << "client disconnected (disconnection handler)" << std::endl;
    should_exit = true;
  });

  client.send({"SET", "hello", "world"}, [] (cpp_redis::reply& reply) {
    std::cout << reply.as_string() << std::endl;
  });
  client.send({"GET", "hello"}, [] (cpp_redis::reply& reply) {
    std::cout << reply.as_string() << std::endl;
  });
  client.commit();

  signal(SIGINT, &sigint_handler);
  while (not should_exit);

  return 0;
}
```

## Redis Subscriber

### Methods

#### void connect(const std::string& host = "127.0.0.1", unsigned int port = 6379, const disconnection_handler& handler = nullptr)
Connect to the Redis Server. Connection is done synchronously.
Throws redis_error in case of failure or if client if already connected.

Also set the disconnection handler which is called whenever a disconnection has occurred.
Disconnection handler is an `std::function<void(redis_subscriber&)>`.

#### void disconnect(void)
Disconnect client from remote host.
Throws redis_error if client is not connected to any server.

#### bool is_connected(void)
Returns whether the client is connected or not.

#### redis_subscriber& subscribe(const std::string& channel, const subscribe_callback& callback)
Subscribe to the given channel and call subscribe_callback each time a message is published in this channel.
subscribe_callback is an `std::function<void(const std::string&, const std::string&)>`.

The command is not effectively sent immediately, but stored inside an internal buffer until `commit()` is called.

#### redis_subscriber& psubscribe(const std::string& pattern, const subscribe_callback& callback)
PSubscribe to the given pattern and call subscribe_callback each time a message is published in a channel matching the pattern.
subscribe_callback is an `std::function<void(const std::string&, const std::string&)>`.

The command is not effectively sent immediately, but stored inside an internal buffer until `commit()` is called.

#### redis_subscriber& unsubscribe(const std::string& channel)
Unsubscribe from the given channel.

The command is not effectively sent immediately, but stored inside an internal buffer until `commit()` is called.

#### redis_subscriber& punsubscribe(const std::string& pattern)
Unsubscribe from the given pattern.

The command is not effectively sent immediately, but stored inside an internal buffer until `commit()` is called.

### redis_subscriber& commit(void)
Send all the commands that have been stored by calling `send()` since the last `commit()` call to the redis server.

That is, pipelining is supported in a very simple and efficient way: `sub.subscribe(...).psubscribe(...).unsubscribe(...).commit()` will send the 3 commands at once (instead of sending 3 network requests, one for each command, as it would have been done without pipelining).

### Example

```cpp
#include <cpp_redis/cpp_redis>

#include <signal.h>
#include <iostream>

volatile std::atomic_bool should_exit(false);
cpp_redis::redis_subscriber sub;

void
sigint_handler(int) {
  std::cout << "disconnected (sigint handler)" << std::endl;
  sub.disconnect();
  should_exit = true;
}

int
main(void) {
  sub.connect("127.0.0.1", 6379, [](cpp_redis::redis_subscriber&) {
    std::cout << "sub disconnected (disconnection handler)" << std::endl;
    should_exit = true;
  });

  sub.subscribe("some_chan", [] (const std::string& chan, const std::string& msg) {
    std::cout << "MESSAGE " << chan << ": " << msg << std::endl;
  });
  sub.psubscribe("*", [] (const std::string& chan, const std::string& msg) {
    std::cout << "PMESSAGE " << chan << ": " << msg << std::endl;
  });
  sub.commit();

  signal(SIGINT, &sigint_handler);
  while (not should_exit);

  return 0;
}
```

## Examples
Some examples are provided in this repository:
* [redis_client.cpp](examples/redis_client.cpp) shows how to use the redis client class.
* [redis_subscriber.cpp](examples/redis_subscriber.cpp) shows how to use the redis subscriber class.

## Author
[Simon Ninon](http://simon-ninon.fr)
