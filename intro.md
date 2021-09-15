# Intro to openpilot

openpilot is a software that produces serial messages (CAN) to accelerate and steer a car given some camera streams, existing serial messages from the car, and sensor data. In addition to the real-time messages, openpilot produces logs that are used to train machine learning models at a later date.

The goal of this introduction is to introduce you to understand the moving pieces of openpilot, and help you understand how they work together to create these outputs.

![conceptual_schematic](https://raw.githubusercontent.com/barbinbrad/openpilot-101/master/conceptual_schematic.png)

## Talking the Car: 101

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
panda->can_send(event.getSendcan());
```
panda->can_send(event.getSendcan());

The boardd process subscribes to `sendcan` topic and turns messages Cap'n Proto messages into CAN messages, using the Panda firmware, which is wired directly to the vehicles, CAN network with a vehicle-specific wiring harness.

## Architecture: 101

![pub_sub](https://raw.githubusercontent.com/barbinbrad/openpilot-101/master/pub_sub.png)
