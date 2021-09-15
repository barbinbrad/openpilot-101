# Intro to openpilot

openpilot is a software that produces serial messages (CAN) to accelerate and steer a car given some camera streams, existing serial messages from the car, and sensor data. In addition to the real-time messages, openpilot produces logs that are used to train machine learning models at a later date.

The goal of this introduction is to introduce you to the moving pieces of openpilot, and help you understand how they work together to create these outputs.

![conceptual_schematic](https://raw.githubusercontent.com/barbinbrad/openpilot-101/master/conceptual_schematic.png)

## Talking to the Car: 101

Modern vehicles communicate using CAN messaging, a broadcast protocol that allows many computers to talk together in a way that is tolerant to noisy environments. From the openpilot perspective, the good thing about the CAN protocol is that it is inherently trusting, allowing messages to be spoofed. The bad thing about the CAN protocol is that each manufacturer creates their own dictionary of CAN message IDs and data definition. 

So even after openpilot has figured what the acceleration and steering angle should be, there is still much work to be done to turn it into a CAN message that will be understood by that particular make/model vehicle. 


```python
# selfdrive/controls/controlsd.py

# send car controls over can
can_sends = CI.apply(CC)
pm.send('sendcan', can_list_to_can_capnp(can_sends))
```

Here is the heart of the code. On each loop of the controlsd process, the car control message, `CC`, represents the desired setpoints of the car (acceleration, steering angle, etc). The car interface, `CI.apply`, is the method for turning these setpoints into make/model specific CAN messages. 

For example, one manufacturer might put the steering angle on message id 0x30 at bytes 3 and 4, while another manufacturer describes steering angle on message id 0xe4 at byte 2 and 3. 

On the next line, `pm.send()` publishes the `can_send` messages on the `sendcan` topic, in Cap'n Proto format. 

```cpp
# selfdrive/boardd/boardd.cc

while (panda->connected) {
    Message * msg = subscriber->receive();

    capnp::FlatArrayMessageReader cmsg(aligned_buf.align(msg));
    cereal::Event::Reader event = cmsg.getRoot<cereal::Event>();

    panda->can_send(event.getSendcan());

    delete msg;
}
```

The boardd process subscribes to the `sendcan` topic and turns Cap'n Proto messages into CAN messages, using the [panda](https://github.com/commaai/panda) hardware and firmware. The `can_send` method uses the Comma hardware's microcontroller and CAN transceivers to send CAN messages directly to the vehicles CAN network. 

## Architecture: 101

So far, we've mentioned two processes: controlsd and boardd. You may have noticed that processes end with the letter "d". That's a tribute to linux background daemons. This section will give a brief overview of the processes and how they communicate with each other.



![pub_sub](https://raw.githubusercontent.com/barbinbrad/openpilot-101/master/pub_sub.png)
