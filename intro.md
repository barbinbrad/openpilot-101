# Intro to openpilot

**openpilot** is a software that produces serial messages (CAN Messages) to change the acceleration and steering angle of a car given some camera streams, existing serial messages from the car, and sensor data. In addition to the real-time messages, openpilot produces logs that are used to train machine learning models at a later date.

The goal of this introduction is to introduce you to the moving pieces of openpilot, and help you understand how they work together to create these outputs. To understand how new cars are added to openpilot, check out energee's [blog post](https://medium.com/@energee/add-support-for-your-car-to-comma-ai-openpilot-3d2da8c12647). 

![conceptual_schematic](https://raw.githubusercontent.com/barbinbrad/openpilot-101/master/conceptual_schematic.png)

## CAN Messages

Modern vehicles communicate using CAN messaging, a broadcast protocol that allows many computers to talk together in a way that is tolerant to noisy environments. From the openpilot perspective, the good thing about the CAN protocol is that it is inherently trusting, allowing messages to be spoofed. The bad thing about the CAN protocol is that each manufacturer creates their own dictionary of CAN message IDs and data definition. 

For example, one manufacturer might put the steering angle on message ID 0x30 at bytes 3 and 4, while another manufacturer describes steering angle on message ID 0xe4 at byte 2 and 3.

So even after openpilot has figured what the acceleration and steering angle should be, there is still much work to be done to turn it into a CAN message that will be understood by that particular make/model vehicle. 

Below is the heart of the code for turning calculations, `CC`, into manufacturer-specific CAN messages using the car interface, `CI`, which contains information about the make/model of the car:


```python
# selfdrive/controls/controlsd.py

# send car controls over can
can_sends = CI.apply(CC)
pm.send('sendcan', can_list_to_can_capnp(can_sends))
```

On each loop of the **controlsd** process, the car control message, `CC`, represents the desired setpoints of the car (acceleration, steering angle, etc). The  `CI.apply` method turns these setpoints into make/model specific CAN messages. 

On the next line, `pm.send()` publishes the `can_send` messages on the `sendcan` topic, in Cap'n Proto format. The next snippet of code describes what happens to the message next. Don't worry if it doesn't make sense yet. 


```cpp
// selfdrive/boardd/boardd.cc

// subscribe to the sendcan topic
SubSocket * subscriber = SubSocket::create(context, "sendcan");

...

while (panda->connected) {
    // get messages as fast as possible
    Message * msg = subcriber->receive();

    // parse the cap'n proto messages
    cereal::Event::Reader event = cmsg.getRoot<cereal::Event>();

    // send the unpacked CAN messages using the panda interface
    panda->can_send(event.getSendcan());
}
```

The **boardd** process subscribes to the `sendcan` topic and turns Cap'n Proto messages into physical CAN messages, through the [panda](https://github.com/commaai/panda) hardware and firmware. The `panda->can_send` method writes to the hardware's microcontroller, which use CAN transceivers to send CAN messages directly to the vehicles CAN network.

To recap: **controlsd** is a process that takes many inputs and turns them into a list of CAN message for a particular make/model vehicle, and **boardd** is a process that talks to the vehicle through the panda interface to send (and receive) physical CAN messages. The two proccesses communicate with each other using a pub-sub messaging framework called [cereal](https://github.com/commaai/panda) where `pm` denotes that a process is a topic publisher, and `sm` denotes that a process is a topic subscriber. 

This separation of concerns into purposeful processes that communicate through pub-sub messaging will be covered in greater detail in the next section.

## Inter-Process Communication

So far, we've mentioned two processes: **controlsd** and **boardd**. This section will give a brief overview of the major processes and how they communicate with each other. You may have noticed that processes all end with the letter d. That's a nod to linux daemons, background processes that have no user control. Similarly, openpilot's processes run without any direct user intervention. 

- **camerad**: interfaces with available camera hardware to generate images, which are encoded as videos 
- **sensorsd**: interfaces with available sensor hardware to generate data from the hardware's IMU
- **boardd**: interfaces with vehicle's CAN network and panda's ublox GPS chip to send and generate
- **ubloxd**: converts raw GPS messages into realtime GPS data
- **locationd**: combines inputs from sensors, gps, and model into realtime estimates for movement and position
- **modeld**: transforms camera images into estimates for movement, future position, and the location of other vehicles
- **plannerd**: [TODO](#)
- **controlsd**: combines a wide range of inputs to produce a car-specific messages for a desired future state 
- **paramsd**: [TODO](#)
- **thermald**: monitors the vitals of the hardware running openpilot
- **loggerd**: logs and uploads messages history and videos
- **athenad**: opens a websocket connection so that message history and videos can be uploaded to the cloud

In the openpilot process map below, processes are represented by nodes, and the topics that they publish-to and subscribe-to are represented by the arrows between the nodes. A process is a publisher of a topic if the arrow points away from the process and is a subscriber if the arrow points toward the process. So, for example, the **thermald** proccess publishes `deviceState` and subscribes to `managerState`, `pandaState`, and `gpsLocationExternal`.

![pub_sub](https://raw.githubusercontent.com/barbinbrad/openpilot-101/master/pub_sub.png)

In the cereal pub-sub model, any process can subscribe to any topic, but each topic can only have one publisher. Messages are stored and exchanged in [Cap'n Proto](https://capnproto.org/) format, an extremely fast and lightweight way to send objects (like JSON) as binary (like ProtoBufs). The structure of every message is contained in the [log.capnp](https://github.com/commaai/cereal/blob/master/log.capnp) file in the cereal sub-module.


## Cereal

In the log.capnp file, a single `Event` struct contains all the topic message definitions:

```capnp
# cereal/log.capnp

struct Event {
  logMonoTime @0 :UInt64;  # nanoseconds
  ...
  union {
    ...
    # ********** openpilot daemon msgs **********
    can @5 :List(CanData);
    controlsState @7 :ControlsState;
    sensorEvents @11 :List(SensorEventData);
    pandaState @12 :PandaState;
    radarState @13 :RadarState;
    liveTracks @16 :List(LiveTracks);
    sendcan @17 :List(CanData);
    liveCalibration @19 :LiveCalibrationData;
    carState @22 :Car.CarState;
    ...
  }
}
```
Notice how both `can` and `sendcan` use the type `List(CanData)`. `List` is a primative type to Cap'n Proto, and `CanData` is a struct that's also defined in log.capnp. The `CanData` struct is composed exclusively of primative types, so no further definition is needed:

```capnp
# cereal/log.capnp

struct CanData {
  address @0 :UInt32; # 11 or 29-bit address
  busTime @1 :UInt16; 
  dat     @2 :Data; # up to 8 bytes of data
  src     @3 :UInt8; # 0-3 or 128-131
}
```

In python or C++, accessing a property `foo` from Cap'n Proto structs is as easy as calling `getFoo()`. Each struct has a `Reader` class that is read-only, and a `Builder` class that is writable. So, for example, to get the top level `Event` struct, we called:

```cpp
// selfdrive/boardd/boardd.cc

// get the root struct
cereal::Event::Reader event = cmsg.getRoot<cereal::Event>();
// pass the sendcan struct to the panda->can_send method
panda->can_send(event.getSendcan());
```

Calling `event.getSendcan()` therefore returns a `List<cereal::CanData>::Reader`, but calling `getFoo()` on a primative type, returns a primative value.

```cpp
// selfdrive/boardd/panda.cc

void Panda::can_send(capnp::List<cereal::CanData>::Reader can_data_list) {
    
    const int msg_count = can_data_list.size();
    for (int i = 0; i < msg_count; i++) {
        // access structs in a list
        auto cmsg = can_data_list[i];
        // get the properties of the struct
        uint32_t address = cmsg.getAddress();
        uint8_t src = cmsg.getSrc();
    }
    ...
    usb_bulk_write(3, (unsigned char*)send.data(), send.size(), 5);
}
```

Publishing a cap'n'proto messages is even easier. Here we see how **paramsd** uses cereal's messaging library to publish a message  on the `liveParameters` topic every time it receives a message on the `liveLocationKalman` topic:

```python
# selfdrive/locationd/paramsd.py
import cereal.messaging as messaging

# subscribe to a list of topics
sm = messaging.SubMaster(['liveLocationKalman', 'carState'], poll=['liveLocationKalman'])
# publish a list of topics
pm = messaging.PubMaster(['liveParameters'])

while True:
    # when the liveLocationKalman receives a new message
    if sm.updated['liveLocationKalman']:
      
      msg = messaging.new_message('liveParameters')
      msg.logMonoTime = sm.logMonoTime['carState']
      ...
     
      msg.liveParameters.posenetValid = True
      msg.liveParameters.sensorValid = True
      msg.liveParameters.steerRatio = float(x[States.STEER_RATIO])
      msg.liveParameters.stiffnessFactor = float(x[States.STIFFNESS])
      msg.liveParameters.angleOffsetAverageDeg = angle_offset_average
      msg.liveParameters.angleOffsetDeg = angle_offset    
       msg.liveParameters.valid = all((
        abs(msg.liveParameters.angleOffsetAverageDeg) < 10.0,
        abs(msg.liveParameters.angleOffsetDeg) < 10.0,
        0.2 <= msg.liveParameters.stiffnessFactor <= 5.0,
        min_sr <= msg.liveParameters.steerRatio <= max_sr,
      ))  
      ....
      pm.send('liveParameters', msg)
```

Notice how the Cap'n Proto struct from cereal matches the message properties, but not all properties are required, such as `gyroBasis`. In the case that properties are not defined, they will be set to 0:

```capnpn
# cereal/log.capnp

liveParameters @61 :LiveParametersData;

...

struct LiveParametersData {
  valid @0 :Bool;
  gyroBias @1 :Float32; #
  angleOffsetDeg @2 :Float32;
  angleOffsetAverageDeg @3 :Float32;
  stiffnessFactor @4 :Float32;
  steerRatio @5 :Float32;
  sensorValid @6 :Bool;
  yawRate @7 :Float32;
  posenetSpeed @8 :Float32;
  posenetValid @9 :Bool;
}

```

Now that we understand what the major processes are and how the talk to each other, lets take a look at how data is persisted for time travel debugging and machine learning.

## Logs

<!--https://github.com/commaai/openpilot/blob/377fe84948c069c3f00824c06e0ae40f2aa2616e/selfdrive/loggerd/loggerd.cc#L352-->

## Panda


## Kalman Filters & PID Loops


## Machine Learning


## Fingerprinting




 

