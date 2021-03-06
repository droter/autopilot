PX4 autopilot
=============

Python autopilot using the PX4 firmware toolchain, Gazebo, ROS, MAVROS and the MAVLink protocol.

## Installation

### Setting up the toolchain
Install the PX4 toolchain, ROS and Gazebo, using the
[PX4 scripts](https://dev.px4.io/master/en/setup/dev_env_linux_ubuntu.html).
You need to execute the `ubuntu_sim_ros_melodic.sh` script to install ROS Melodic and MavROS.
Then, execute the `ubuntu.sh` script to install simulators like Gazebo and the rest
of the PX4 toolchain. You **need** to use Ubuntu 18, otherwise it will not work !

You may need to clone the Firmware. We cloned it in the home folder.

The source code is in `~/catkin_ws/src/`, with some common modules like MAVROS,
and the custom autopilot from this repository. You need to always build the packages
with `catkin build`.

In the `.bashrc`, you need the following lines. Adapt `$fw_path` and `$gz_path` according to the location
of the PX4 firmware.
```shell script
source /opt/ros/melodic/setup.bash
source ~/catkin_ws/devel/setup.bash

fw_path="$HOME/Firmware"
gz_path="$fw_path/Tools/sitl_gazebo"
source $fw_path/Tools/setup_gazebo.bash $fw_path $fw_path/build/px4_sitl_default
export ROS_PACKAGE_PATH=$ROS_PACKAGE_PATH:$fw_path
export ROS_PACKAGE_PATH=$ROS_PACKAGE_PATH:$gz_path

# Set the plugin path so Gazebo finds our model and sim
export GAZEBO_PLUGIN_PATH=${GAZEBO_PLUGIN_PATH}:$gz_path/build
# Set the model path so Gazebo finds the airframes
export GAZEBO_MODEL_PATH=${GAZEBO_MODEL_PATH}:$gz_path/models
# Disable online model lookup since this is quite experimental and unstable
export GAZEBO_MODEL_DATABASE_URI=""
export SITL_GAZEBO_PATH=$gw_path
```

### Compile and run a vanilla simulation

First, you need to compile the PX4 toolchain once.
```shell script
DONT_RUN=1 make px4_sitl_default gazebo
```

We use `roslaunch` to launch every ROS nodes necessary.
```shell script
roslaunch px4 mavros_posix_sitl.launch
```

This command launch the launch file located in the PX4 ROS package in
`Firmware/launch/mavros_posix_sitl.launch`.
It launches PX4 as SITL, MAVROS, Gazebo connected to PX4, and spawns the UAV.

It is possible to send some arguments to change the vehicle initial pose,
the world or the vehicle type.
```shell script
roslaunch px4 mavros_posix_sitl.launch x:=10 y:=10 world:=$HOME/Firmware/Tools/sitl_gazebo/worlds/warehouse.world
```
The maps are stored in the PX4 toolchain in `Firmware/Tools/sitl_gazebo/worlds/`.
The Gazebo models and UAVs used in the simulation are in `Firmware/Tools/sitl_gazebo/models/`.

### Setting up the octomap

To use the octomap generated by all the sensors, you need to install the
`octomap_serveur` node to be used in ROS.
```shell script
sudo apt install ros-melodic-octomap ros-melodic-octomap-mapping
rosdep install octomap_mapping
rosmake octomap_mapping
```

Then, in order to use the octomap data in the autopilot, you need the
[python wrapper](https://github.com/wkentaro/octomap-python)
of the [C++ octomap library](https://github.com/OctoMap/octomap).
You can install it with `pip2`, but I had conflicts between python 2 and 3, so
I compiled the module directly.
```shell script
git clone --recursive https://github.com/wkentaro/octomap-python.git
cd octomap-python
python2 setup.py build
python2 setup.py install
```

If you are using the _ground truth_ octomap of the Gazebo world,
you need to generate the `.bt` file representation of the octomap.
You can generate it using the `world2oct.sh` script in the `autopilot/world_to_octomap/`
folder of this repo.

For more info, refer to `autopilot/world_to_octomap/readme.md`.

## Run the autopilot

To launch all necessary nodes, a launch file is available in `autopilot/launch/autopilot.launch`.
It launches ROS, MAVROS, PX4, Gazebo and the Octomap server using a `.bt` file.
You can also specify a vehicle, and a starting position.
```shell script
roslaunch autopilot autopilot.launch octomap:=octomaps/warehouse.bt vehicle:=iris world:=warehouse
```

To visualize the octomap, you can use `rosrun rviz rviz`.

**TODO**: The launch file should start the autopilot node.
The autopilot will communicate with the MAVROS topics to move the UAV.

## How to use the autopilot

```shell script
rosrun autopilot main.py -p <planning_algorithm>
```

`<planning_algorithm>` can be:
- A*
- ~~RRT~~
- RRT*
- dummy (Dummy planning algorithm, for testing purposes)

The autopilot path planning compute a list of waypoints to local coordinate system,
which it converts to GPS coordinates and builds a MAVLink mission. The mission
is published to a MAVROS topic and the flight mode is set to `AUTO.MISSION`, so
the FCU execute the mission loaded.

## DEPRECATED

This section is for commands not useful for the project.

To launch PX4, using the Gazebo simulation, but **without ROS and MAVROS**. We use the quadcopter **iris** and the map **warehouse**.
```shell script
make px4_sitl_default gazebo_iris__warehouse
```

To install the C++ Octomap library system wide.
[Reference](https://github.com/OctoMap/octomap/wiki/Compilation-and-Installation-of-OctoMap)

```shell script
git clone git://github.com/OctoMap/octomap.git
cd octomap/octomap
mkdir build
cmake ..
make
make test # To verify the build
make install # To install the library system wide
```
