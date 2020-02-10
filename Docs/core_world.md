<h1>World</h1>

#### Connecting and retrieving the world

To connect to a simulator we need to create a "Client" object, to do so we need
to provide the IP address and port of a running instance of the simulator

```py
client = carla.Client('localhost', 2000)
```

The first recommended thing to do right after creating a client instance is
setting its time-out. This time-out sets a time limit to all networking
operations, if the time-out is not set networking operations may block forever

```py
client.set_timeout(10.0) # seconds
```

Once we have the client configured we can directly retrieve the world

```py
world = client.get_world()
```

Typically we won't need the client object anymore, all the objects created by
the world will connect to the IP and port provided if they need to. These
operations are usually done in the background and are transparent to the user.

#### Changing the weather

The lighting and weather conditions can be requested and changed with the world
object

```py
weather = carla.WeatherParameters(
    cloudiness=80.0,
    precipitation=30.0,
    sun_altitude_angle=70.0)

world.set_weather(weather)

print(world.get_weather())
```

For convenience, we also provided a list of predefined weather presets that can
be directly applied to the world

```py
world.set_weather(carla.WeatherParameters.WetCloudySunset)
```

The full list of presets can be found in the
[WeatherParameters reference](python_api.md#carla.WeatherParameters).

### World Snapshot

A world snapshot represents the state of every actor in the simulation at a single frame,
a sort of still image of the world with a timestamp. With this feature it is possible to
record the location of every actor and make sure all of them were captured at the same
frame without the need of using synchronous mode.

```py
# Retrieve a snapshot of the world at this point in time.
world_snapshot = world.get_snapshot()

# Wait for the next tick and retrieve the snapshot of the tick.
world_snapshot = world.wait_for_tick()

# Register a callback to get called every time we receive a new snapshot.
world.on_tick(lambda world_snapshot: do_something(world_snapshot))
```

The world snapshot contains a timestamp and a list of actor snapshots. Actor snapshots do not
allow to operate on the actor directly as they only contain data about the physical state of
the actor, but you can use their id to retrieve the actual actor. And the other way around,
you can look up snapshots by id (average O(1) complexity).

```py
timestamp = world_snapshot.timestamp
timestamp.frame_count
timestamp.elapsed_seconds
timestamp.delta_seconds
timestamp.platform_timestamp


for actor_snapshot in world_snapshot:
    actor_snapshot.get_transform()
    actor_snapshot.get_velocity()
    actor_snapshot.get_angular_velocity()
    actor_snapshot.get_acceleration()

    actual_actor = world.get_actor(actor_snapshot.id)


actor_snapshot = world_snapshot.find(actual_actor.id)
```