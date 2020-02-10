<h1>Actors</h1>


#### Blueprints

A blueprint contains the information necessary to create a new actor. For
instance, if the blueprint defines a car, we can change its color here, if it
defines a lidar, we can decide here how many channels the lidar will have. A
blueprints also has an ID that uniquely identifies it and all the actor
instances created with it. Examples of IDs are "vehicle.nissan.patrol" or
"sensor.camera.depth".

The list of all available blueprints is kept in the [**blueprint library**](/bp_library)

```py
blueprint_library = world.get_blueprint_library()
```

The library allows us to find specific blueprints by ID, filter them with
wildcards, or just choosing one at random

```py
# Find specific blueprint.
collision_sensor_bp = blueprint_library.find('sensor.other.collision')
# Chose a vehicle blueprint at random.
vehicle_bp = random.choice(blueprint_library.filter('vehicle.bmw.*'))
```

Some of the attributes of the blueprints can be modified while some other are
just read-only. For instance, we cannot modify the number of wheels of a vehicle
but we can change its color

```py
vehicles = blueprint_library.filter('vehicle.*')
bikes = [x for x in vehicles if int(x.get_attribute('number_of_wheels')) == 2]
for bike in bikes:
    bike.set_attribute('color', '255,0,0')
```

Modifiable attributes also come with a list of recommended values

```py
for attr in blueprint:
    if attr.is_modifiable:
        blueprint.set_attribute(attr.id, random.choice(attr.recommended_values))
```

The blueprint system has been designed to ease contributors adding their custom
actors directly in Unreal Editor, we'll add a tutorial on this soon, stay tuned!

#### Spawning actors

Once we have the blueprint set up, spawning an actor is pretty straightforward

```py
transform = Transform(Location(x=230, y=195, z=40), Rotation(yaw=180))
actor = world.spawn_actor(blueprint, transform)
```

The spawn actor function comes in two flavours, [`spawn_actor`](python_api.md#carla.World.spawn_actor) and
[`try_spawn_actor`](python_api.md#carla.World.try_spawn_actor).
The former will raise an exception if the actor could not be spawned,
the later will return `None` instead. The most typical cause of
failure is collision at spawn point, meaning the actor does not fit at the spot
we chose; probably another vehicle is in that spot or we tried to spawn into a
static object.

To ease the task of finding a spawn location, each map provides a list of
recommended transforms

```py
spawn_points = world.get_map().get_spawn_points()
```

We'll add more on the map object later in this tutorial.

Finally, the spawn functions have an optional argument that controls whether the
actor is going to be attached to another actor. This is specially useful for
sensors. In the next example, the camera remains rigidly attached to our vehicle
during the rest of the simulation

```py
camera = world.spawn_actor(camera_bp, relative_transform, attach_to=my_vehicle)
```

Note that in this case, the transform provided is treated relative to the parent
actor.

#### Handling actors

Once we have an actor alive in the world, we can move this actor around and
check its dynamic properties

```py
location = actor.get_location()
location.z += 10.0
actor.set_location(location)
print(actor.get_acceleration())
print(actor.get_velocity())
```

We can even freeze an actor by disabling its physics simulation

```py
actor.set_simulate_physics(False)
```

And once we get tired of an actor we can remove it from the simulation with

```py
actor.destroy()
```

Note that actors are not cleaned up automatically when the Python script
finishes, if we want to get rid of them we need to explicitly destroy them.

!!! important
    **Known issue:** To improve performance, most of the methods send requests
    to the simulator asynchronously. The simulator queues each of these
    requests, but only has a limited amount of time each update to parse them.
    If we flood the simulator by calling "set" methods too often, e.g.
    set_transform, the requests will accumulate a significant lag.


#### Sensors

Sensors are actors that produce a stream of data. Sensors are such a key
component of CARLA that they deserve their own documentation page, so here we'll
limit ourselves to show a small example of how sensors work

```py
camera_bp = blueprint_library.find('sensor.camera.rgb')
camera = world.spawn_actor(camera_bp, relative_transform, attach_to=my_vehicle)
camera.listen(lambda image: image.save_to_disk('output/%06d.png' % image.frame))
```

In this example we have attached a camera to a vehicle, and told the camera to
save to disk each of the images that are going to be generated.

The full list of sensors and their measurement is explained in
[Cameras and sensors](cameras_and_sensors.md).

#### Vehicles

Vehicles are a special type of actor that provide a few extra methods. Apart
from the handling methods common to all actors, vehicles can also be controlled
by providing throttle, break, and steer values

```py
vehicle.apply_control(carla.VehicleControl(throttle=1.0, steer=-1.0))
```

These are all the parameters of the [`VehicleControl`](python_api.md#carla.VehicleControl)
object and their default values

```py
carla.VehicleControl(
    throttle = 0.0
    steer = 0.0
    brake = 0.0
    hand_brake = False
    reverse = False
    manual_gear_shift = False
    gear = 0)
```

Also, physics control properties can be tuned for vehicles and its wheels

```py
vehicle.apply_physics_control(carla.VehiclePhysicsControl(max_rpm = 5000.0, center_of_mass = carla.Vector3D(0.0, 0.0, 0.0), torque_curve=[[0,400],[5000,400]]))
```

These properties are controlled through a 
[`VehiclePhysicsControl`](python_api.md#carla.VehiclePhysicsControl) object,
which also contains a property to control each wheel's physics through a
[`WheelPhysicsControl`](python_api.md#carla.WheelPhysicsControl) object.

```py
carla.VehiclePhysicsControl(
    torque_curve,
    max_rpm,
    moi,
    damping_rate_full_throttle,
    damping_rate_zero_throttle_clutch_engaged,
    damping_rate_zero_throttle_clutch_disengaged,
    use_gear_autobox,
    gear_switch_time,
    clutch_strength,
    mass,
    drag_coefficient,
    center_of_mass,
    steering_curve,
    wheels)
```

Where:

- *torque_curve*: Curve that indicates the torque measured in Nm for a specific revolutions
per minute of the vehicle's engine
- *max_rpm*: The maximum revolutions per minute of the vehicle's engine
- *moi*: The moment of inertia of the vehicle's engine
- *damping_rate_full_throttle*: Damping rate when the throttle is maximum.
- *damping_rate_zero_throttle_clutch_engaged*: Damping rate when the thottle is zero
with clutch engaged
- *damping_rate_zero_throttle_clutch_disengaged*: Damping rate when the thottle is zero
with clutch disengaged

- *use_gear_autobox*: If true, the vehicle will have automatic transmission
- *gear_switch_time*: Switching time between gears
- *clutch_strength*: The clutch strength of the vehicle. Measured in Kgm^2/s

- *final_ratio*: The fixed ratio from transmission to wheels.
- *forward_gears*: List of [`GearPhysicsControl`](python_api.md#carla.GearPhysicsControl) objects.

- *mass*: The mass of the vehicle measured in Kg
- *drag_coefficient*: Drag coefficient of the vehicle's chassis
- *center_of_mass*: The center of mass of the vehicle
- *steering_curve*: Curve that indicates the maximum steering for a specific forward speed
- *wheels*: List of [`WheelPhysicsControl`](python_api.md#carla.WheelPhysicsControl) objects.

```py
carla.WheelPhysicsControl(
    tire_friction,
    damping_rate,
    max_steer_angle,
    radius,
    max_brake_torque,
    max_handbrake_torque,
    position)
```
Where:
- *tire_friction*: Scalar value that indicates the friction of the wheel.
- *damping_rate*: The damping rate of the wheel.
- *max_steer_angle*: The maximum angle in degrees that the wheel can steer.
- *radius*: The radius of the wheel in centimeters.
- *max_brake_torque*: The maximum brake torque in Nm.
- *max_handbrake_torque*: The maximum handbrake torque in Nm.
- *position*: The position of the wheel.

```py
carla.GearPhysicsControl(
    ratio,
    down_ratio,
    up_ratio)
```
Where:
- *ratio*: The transmission ratio of this gear.
- *down_ratio*: The level of RPM (in relation to MaxRPM) where the gear autobox initiates shifting down.
- *up_ratio*: The level of RPM (in relation to MaxRPM) where the gear autobox initiates shifting up.

Our vehicles also come with a handy autopilot

```py
vehicle.set_autopilot(True)
```

As has been a common misconception, we need to clarify that this autopilot
control is purely hard-coded into the simulator and it's not based at all in
machine learning techniques.

Finally, vehicles also have a bounding box that encapsulates them

```py
box = vehicle.bounding_box
print(box.location)         # Location relative to the vehicle.
print(box.extent)           # XYZ half-box extents in meters.
```

#### Walkers

![pedestrian types](img/pedestrian_types.png)

We can get a lit of all pedestrians from the blueprint library and choose one:

```py
world = client.get_world()
blueprintsWalkers = world.get_blueprint_library().filter("walker.pedestrian.*")
walker_bp = random.choice(blueprintsWalkers)
```

We can **get a list of random points** where to spawn the pedestrians. Those points are always
from the areas where the pedestrian can walk:

```py
# 1. take all the random locations to spawn
spawn_points = []
for i in range(50):
    spawn_point = carla.Transform()
    spawn_point.location = world.get_random_location_from_navigation()
    if (spawn_point.location != None):
        spawn_points.append(spawn_point)

```

Now we can **spawn the pedestrians** at those positions using a batch of commands:

```py
# 2. build the batch of commands to spawn the pedestrians
batch = []
for spawn_point in spawn_points:
    walker_bp = random.choice(blueprintsWalkers)
    batch.append(carla.command.SpawnActor(walker_bp, spawn_point))

# apply the batch
results = client.apply_batch_sync(batch, True)
for i in range(len(results)):
    if results[i].error:
        logging.error(results[i].error)
    else:
        walkers_list.append({"id": results[i].actor_id})
```

We save the id of each walker from the results of the batch, in a dictionary because we will
assign to them also a controller.
We need to **create the controller** that will manage the pedestrian automatically:

```py
# 3. we spawn the walker controller
batch = []
walker_controller_bp = world.get_blueprint_library().find('controller.ai.walker')
for i in range(len(walkers_list)):
    batch.append(carla.command.SpawnActor(walker_controller_bp, carla.Transform(), walkers_list[i]["id"]))

# apply the batch
results = client.apply_batch_sync(batch, True)
for i in range(len(results)):
    if results[i].error:
        logging.error(results[i].error)
    else:
        walkers_list[i]["con"] = results[i].actor_id
```

We create the controller as child of the walker, so the location we pass is (0,0,0).

At this point we have a list of pedestrians with a controller each one, but we need to get
the actual actor from the id. Because the controller is a child of the pedestrian,
we need to **put all id in the same list** so the parent can find the child in the same list.

```py
# 4. we put altogether the walkers and controllers id to get the objects from their id
for i in range(len(walkers_list)):
    all_id.append(walkers_list[i]["con"])
    all_id.append(walkers_list[i]["id"])
all_actors = world.get_actors(all_id)
```

The list all_actors has now all the actor objects we created.

At this point is a good idea to **wait for a tick** on client, because then the server has
time to send all new data about the new actors we just created (we need the transform of
each one updated). So we can do a call like:

```py
# wait for a tick to ensure client receives the last transform of the walkers we have just created
world.wait_for_tick()
```

After that, our client has the data about the actors updated.

 **Using the controller** we can set the locations where we want each pedestrian walk to:

```py
# 5. initialize each controller and set target to walk to (list is [controller, actor, controller, actor ...])
for i in range(0, len(all_actors), 2):
    # start walker
    all_actors[i].start()
    # set walk to random point
    all_actors[i].go_to_location(world.get_random_location_from_navigation())
    # random max speed
    all_actors[i].set_max_speed(1 + random.random())    # max speed between 1 and 2 (default is 1.4 m/s)
```

There we have set at each pedestrian (through its controller) a random point and random speed.
When they reach the target point then automatically walk to another random point.

If the target point is not reachable, then they reach the closest point from the are where they are.

![pedestrian sample](img/pedestrians_shoot.png)

To **destroy the pedestrians**, we need to stop them from the navigation,
and then destroy the objects (actor and controller):

```py
# stop pedestrians (list is [controller, actor, controller, actor ...])
for i in range(0, len(all_id), 2):
    all_actors[i].stop()

# destroy pedestrian (actor and controller)
client.apply_batch([carla.command.DestroyActor(x) for x in all_id])
```

#### Other actors

Apart from vehicles and sensors, there are a few other actors in the world. The
full list can be requested to the world with

```py
actor_list = world.get_actors()
```

The actor list object returned has functions for finding, filtering, and
iterating actors

```py
# Find an actor by id.
actor = actor_list.find(id)
# Print the location of all the speed limit signs in the world.
for speed_sign in actor_list.filter('traffic.speed_limit.*'):
    print(speed_sign.get_location())
```

Among the actors you can find in this list are

  * **Traffic lights** with a [`state`](python_api.md#carla.TrafficLight.state) property
  to check the light's current state.
  * **Speed limit signs** with the speed codified in their type_id.
  * The **Spectator** actor that can be used to move the view of the simulator window.