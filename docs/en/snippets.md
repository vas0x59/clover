Code examples
===

Python
---

### # {#distance}

Calculating the distance between two points (**important**: the points are to be in the same [system of coordinates](frames.md)):

```python
def get_distance(x1, y1, z1, x2, y2, z2):
    return math.sqrt((x1 - x2) ** 2 + (y1 - y2) ** 2 + (z1 - z2) ** 2)
```

### # {#distance-global}

Approximation of distance (in meters) between two global coordinates (latitude/longitude):

```python
def get_distance_global(lat1, lon1, lat2, lon2):
    return math.hypot(lat1 - lat2, lon1 - lon2) * 1.113195e5
```

### # {#block-takeoff}

Takeoff and waiting for it to finish:

```python
z = 2  # altitude
tolerance = 0.2 # precision of altitude check (m)

# Saving the initial point
start = get_telemetry()

# Take off and leveling at 2 m above the ground
print navigate(z=z, speed=0.5, frame_id='body', auto_arm=True)

# Waiting for takeoff
while not rospy.is_shutdown():
    # Checking current altitude
    if start.z + z - get_telemetry().z < tolerance:
        # Takeoff complete
        break
    rospy.sleep(0.2)
```

This code can be wrapped in a function:

```python
def takeoff_wait(alt, speed=0.5, tolerance=0.2):
    start = get_telemetry()
    print navigate(z=alt, speed=speed, frame_id='body', auto_arm=True)

    while not rospy.is_shutdown():
        if start.z + alt - get_telemetry().z < tolerance:
            break

        rospy.sleep(0.2)
```

### # {#block-nav}

Flying towards a point and waiting for copter's arrival:

```python
tolerance = 0.2 # precision of arrival check (m)
frame_id='aruco_map'

# Flying to point 1:2:3 in the field of ArUco markers
print navigate(frame_id=frame_id, x=1, y=2, z=3, speed=0.5)

# Wait for the copter to arrive at the requested point
while not rospy.is_shutdown():
    telem = get_telemetry(frame_id=frame_id)
    # Calculating the distance to the requested point
    if get_distance(1, 2, 3, telem.x, telem.y, telem.z) < tolerance:
        # Arrived at the requested point
        break
    rospy.sleep(0.2)
```

This code can be wrapped into a function:

```python
def navigate_wait(x, y, z, speed, frame_id, tolerance=0.2):
    navigate(x=x, y=y, z=z, speed=speed, frame_id=frame_id)

    while not rospy.is_shutdown():
        telem = get_telemetry(frame_id=frame_id)
        if get_distance(x, y, z, telem.x, telem.y, telem.z) < tolerance:
            break
        rospy.sleep(0.2)
```

A more universal solution, utilizing the `navigate_target` frame, which corresponds to the navigating target point of the drone:

```python
def navigate_wait(x, y, z, speed, frame_id, tolerance=0.2):
    navigate(x=x, y=y, z=z, speed=speed, frame_id=frame_id)

    while not rospy.is_shutdown():
        telem = get_telemetry(frame_id='navigate_target')
        if math.sqrt(telem.x ** 2 + telem.y ** 2 + telem.z ** 2) < tolerance:
            break
        rospy.sleep(0.2)
```

This code also can be used for navigating using `body` frame.

### # {#block-land}

Landing and waiting until the copter lands:

```python
land()
while get_telemetry().armed:
    rospy.sleep(0.2)
```

This code can be wrapped in a function:

```python
def land_wait():
    land()
    while get_telemetry().armed:
        rospy.sleep(0.2)
```

### # {#disarm}

Quadcopter disarm (disabling propellers **the copter will fall down**):

```python
# Declaring a proxy:
from mavros_msgs.srv import CommandBool
arming = rospy.ServiceProxy('mavros/cmd/arming', CommandBool)

# ...

arming(False)  # дизарм
```

### # {#transform}

Transforming the position (`PoseStamped`) from one system of coordinates ([of frame](frames.md)) to another one using [tf2] (http://wiki.ros.org/tf2):

```python
import tf2_ros
import tf2_geometry_msgs
from geometry_msgs.msg import PoseStamped

tf_buffer = tf2_ros.Buffer()
tf_listener = tf2_ros.TransformListener(tf_buffer)

# ...

# PoseStamped object creation (or getting it from a topic):
pose = PoseStamped()
pose.header.frame_id = 'map' # frame, which is the position is specified
pose.header.stamp = rospy.get_rostime() # the instant for which the position is specified (current time)
pose.pose.position.x = 1
pose.pose.position.y = 2
pose.pose.position.z = 3
pose.pose.orientation.w = 1

frame_id = 'base_link' # target frame
transform_timeout = rospy.Duration(0.2) # wait for transformation timeout

# Transforming the position from the old frame to the new one:
new_pose = tf_buffer.transform(pose, frame_id, transform_timeout)
```

### # {#upside-down}

Determining whether the copter is turned upside-down:

```python
PI_2 = math.pi / 2
telem = get_telemetry()

flipped = abs(telem.pitch) > PI_2 or abs(telem.roll) > PI_2
```

### # {#angle-hor}

Calculating the copter total horizontal angle:

```python
PI_2 = math.pi / 2
telem = get_telemetry()

flipped = not -PI_2 <= telem.pitch <= PI_2 or not -PI_2 <= telem.roll <= PI_2
angle_to_horizon = math.atan(math.hypot(math.tan(telem.pitch), math.tan(telem.roll)))
if flipped:
    angle_to_horizon = math.pi - angle_to_horizon
```

### # {#circle}

Flying along a circular path:

```python
RADIUS = 0.6  # m
SPEED = 0.3 # rad / s

start = get_telemetry()
start_stamp = rospy.get_rostime()

r = rospy.Rate(10)

while not rospy.is_shutdown():
    angle = (rospy.get_rostime() - start_stamp).to_sec() * SPEED
    x = start.x + math.sin(angle) * RADIUS
    y = start.y + math.cos(angle) * RADIUS
    set_position(x=x, y=y, z=start.z)

    r.sleep()
```

### # {#rate}

repeating an action at a frequency of 10 Hz:

```python
r = rospy.Rate(10)
while not rospy.is_shutdown():
    # Do anything
    r.sleep()
```

### # {#mavros-sub}

An example of subscription to a topic from MAVROS

```python
from geometry_msgs.msg import PoseStamped, TwistStamped
from sensor_msgs.msg import BatteryState
from mavros_msgs.msg import RCIn

# ...

def pose_update(pose):
    # Processing new data of copter's position
    pass

# Other handler functions
# ...

rospy.Subscriber('/mavros/local_position/pose', PoseStamped, pose_update)
rospy.Subscriber('/mavros/local_position/velocity', TwistStamped, velocity_update)
rospy.Subscriber('/mavros/battery', BatteryState, battery_update)
rospy.Subscriber('mavros/rc/in', RCIn, rc_callback)
```

Information about MAVROS topics is available at [the link](mavros.md).

<!-- markdownlint-disable MD044 -->

### # {#mavlink}

<!-- markdownlint-enable MD044 -->

Sending an arbitrary [MAVLink message](mavlink.md) to the copter:

```python
# ...

from mavros_msgs.msg import Mavlink
from mavros import mavlink
from pymavlink import mavutil

# ...

mavlink_pub = rospy.Publisher('mavlink/to', Mavlink, queue_size=1)

# Sending a HEARTBEAT message:

msg = mavutil.mavlink.MAVLink_heartbeat_message(mavutil.mavlink.MAV_TYPE_GCS, 0, 0, 0, 0, 0)
msg.pack(mavutil.mavlink.MAVLink('', 2, 1))
ros_msg = mavlink.convert_to_rosmsg(msg)

mavlink_pub.publish(ros_msg)
```

### # {#rc-sub}

Return on mode switching with the transmitter (may be used for starting an autonomous flight, see [example](https://gist.github.com/okalachev/b709f04522d2f9af97e835baedeb806b)):

```python
from mavros_msgs.msg import RCIn

# Called when new data is received from the transmitter
def rc_callback(data):
    # Return on switch toggling of the transmitter
        if data.channels[5] < 1100:
        # ...
        pass
    elif data.channels[5] > 1900:
        # ...
        pass
    else:
        # ...
        pass

# Creating a subscriber for the topic with the data from the transmitter
rospy.Subscriber('mavros/rc/in', RCIn, rc_callback)

rospy.spin()
```

### # {#set_mode}

Change the [flight mode](modes.md) to arbitrary one:

```python
from mavros_msgs.srv import SetMode

# ...

set_mode = rospy.ServiceProxy('mavros/set_mode', SetMode)

# ...

set_mode(custom_mode='STABILIZED')
```

### # {#flip}

Flip:

```python
import math

# ...

PI_2 = math.pi / 2

def flip():
    start = get_telemetry()  # memorize starting position

    set_rates(thrust=1)  # bump up
    rospy.sleep(0.2)

    set_rates(pitch_rate=30, thrust=0.2)  # pitch flip
    # set_rates(roll_rate=30, thrust=0.2)  # roll flip

    while True:
        telem = get_telemetry()
        flipped = abs(telem.pitch) > PI_2 or abs(telem.roll) > PI_2
        if flipped:
            break

    rospy.loginfo('finish flip')
    set_position(x=start.x, y=start.y, z=start.z, yaw=start.yaw)  # finish flip

print navigate(z=2, speed=1, frame_id='body', auto_arm=True)  # take off
rospy.sleep(10)

rospy.loginfo('flip')
flip()
```

Requires the [special PX4 firmware for Clover](firmware.md#modified-firmware-for-clover). Before running a flip, take all necessary safty precautions.

### # {#calibrate-gyro}

Perform gyro calibration:

```python
from pymavlink import mavutil
from mavros_msgs.srv import CommandLong
from mavros_msgs.msg import State

# ...

send_command = rospy.ServiceProxy('/mavros/cmd/command', CommandLong)

def calibrate_gyro():
    rospy.loginfo('Calibrate gyro')
    if not send_command(command=mavutil.mavlink.MAV_CMD_PREFLIGHT_CALIBRATION, param1=1).success:
        return False

    calibrating = False
    while not rospy.is_shutdown():
        state = rospy.wait_for_message('mavros/state', State)
        if state.system_status == mavutil.mavlink.MAV_STATE_CALIBRATING or state.system_status == mavutil.mavlink.MAV_STATE_UNINIT:
            calibrating = True
        elif calibrating and state.system_status == mavutil.mavlink.MAV_STATE_STANDBY:
            rospy.loginfo('Calibrating finished')
            return True

calibrate_gyro()
```

> **Note** In process of calibration the drone should not be moved.
