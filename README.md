![Data Tamer](data_tamer_logo.png)

**DataTamer** is a spiritual successor of [pal_statistics](https://github.com/pal-robotics/pal_statistics).

Its purpose is to trace **numerical** values in your application and export them into a
format that allows users to visualize them as timeseries, 
for instance in [PlotJuggler](https://github.com/facontidavide/PlotJuggler).

All the values are aggregated in a single "snapshot", for this reason, it is particularly 
suited to record data in a periodic loop (very frequent in robotics applications).

## Features

- **Serialization schema is created at run-time**: no need to do any code generation.
- **Suitable for real-time applications**: very low-latency (on the side of the callee).
- **Multi-sink architecture**: recorded data can be forwarded to multiple "backends". 
- **Very low serialization overhead**.
- The user can enable/disable traced variables at run-time.

Available sinks:

- Direct [MCAP](https://mcap.dev/) recording.
- `DummySink`, mostly useful for debugging and unit testing.
- ROS2 publisher (coming soon). 

## Limitations

- Traced variables can not be added (registered) once the recording starts.
- Can only be used with triavially copiable objects (therefore, no variable size vectors).
- Focused on periodic recording. Not the best option for sporadic, asynchornous events.
- If you use `DataTamer::registerValue` you must be careful about the lifetime of the
object. If you prefer a safer interface, use `DataTamer::createLoggedValue` instead.

# Example

```cpp
#include "data_tamer/data_tamer.hpp"
#include "data_tamer/sinks/dummy_sink.hpp"

int main()
{
  using namespace DataTamer;

  // Start defining one or more Sinks that must be added by default.
  // Do this BEFORE creating a channel.
  auto dummy_sink = std::make_shared<DummySync>();
  ChannelsRegistry::Global().addDefaultSink(dummy_sink);

  // Create a channel (or get an existing one) using the 
  // global registry (singleton-like interface)
  auto channel = ChannelsRegistry::Global().getChannel("my_channel");

  // If you don't want to use addDefaultSink, you can do it manually:
  // channel->addDataSink(dummy_sink);

  // You can register any arithmetic value. You are responsible for their lifetime
  double value_real = 3.14;
  int value_int = 42;
  auto id1 = channel->registerValue("value_real", &value_real);
  auto id2 = channel->registerValue("value_int", &value_int);

  // If you prefer to use RAII, use this method instead
  // logged_real will disable itself when it goes out of scope.
  auto logged_float = channel->createLoggedValue<float>("my_real");

  // This is the way you store the current snapshot of the values
  channel->takeSnapshot();

  // You can disable (i.e., stop recording) a value
  channel->setEnabled(id1, false);
  // or
  logged_float->setEnabled(false);

  // The serialized data of the next snapshot will contain
  // only [value_int], i.e. [id2], since the other two were disabled
  channel->takeSnapshot();
}
```
