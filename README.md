# ROS 2 Basics
This repository contains three ROS 2 examples:
- demo_nodes_cpp (talker/listener)
- turtlesim (GUI + keyboard teleop)
- colcon build workflow for building custom packages

It also includes:
- macOS X11 forwarding setup (XQuartz)
- ROS 2 core concepts and useful CLI commands
- package build workflow with rosdep and colcon

## 1. Build and run the ROS 2 Docker image

### Build the image
From the repository root:

```bash
docker build --build-arg USERNAME=arthurfont -t ros2-minimal -f ros2_ws/Dockerfile .
```

### Start a container

```bash
docker run -it --rm --name ros2-minimal -v "$PWD/ros2_ws:/ros2_ws" ros2-minimal
```

### Verify ROS environment inside the container

```bash
printenv | grep -i ROS
```

## 2. Example 1: `demo_nodes_cpp` (talker/listener)

Inside the running container:

```bash
sudo apt update
sudo apt install -y ros-jazzy-demo-nodes-cpp
ros2 pkg list | grep demo_nodes_cpp
```

Terminal A:

```bash
ros2 run demo_nodes_cpp listener
```

Terminal B:

```bash
docker exec -it ros2-minimal bash
source /opt/ros/jazzy/setup.bash
ros2 run demo_nodes_cpp talker
```

## 3. Example 2: turtlesim (GUI + teleop)

### Terminal 1: start turtlesim

```bash
xhost +localhost
docker run -it --rm --name ros2-minimal --env DISPLAY=host.docker.internal:0 -v "$PWD/ros2_ws:/ros2_ws" ros2-minimal
sudo apt install -y ros-jazzy-turtlesim
ros2 run turtlesim turtlesim_node
```

### Terminal 2: keyboard teleop

```bash
docker exec -it ros2-minimal bash
source /opt/ros/jazzy/setup.bash
ros2 run turtlesim turtle_teleop_key
```

### Terminal 3 (optional): rqt graph and tools

```bash
docker exec -it ros2-minimal bash
source /opt/ros/jazzy/setup.bash
sudo apt install -y '~nros-jazzy-rqt*'
rqt --force-discover
```

### Introspection commands

```bash
ros2 node list
ros2 topic list
ros2 service list
ros2 action list
```

## 4. Example 3: Build workspace using colcon

Get example package source code:

```bash
mkdir -p ros2_ws/src
cd ros2_ws/src/
git clone https://github.com/ros/ros_tutorials.git --depth 1 -b jazzy
rm -rf ros_tutorials/.git
cd ../..
```

Start the container:

```bash
docker run -it --rm --name ros2-minimal --env DISPLAY=host.docker.internal:0 -v "./ros2_ws:/ros2_ws" ros2-minimal
rosdep update
rosdep install -i --from-path ros2_ws/src --rosdistro jazzy -y
cd ros2_ws/
colcon build
```

Before sourcing the overlay, it is very important that you open a new terminal, separate from the one where you built the workspace. In a new terminal, source underlay and overlay:

```bash
docker exec -it ros2-minimal bash
source /opt/ros/jazzy/setup.bash
cd ros2_ws/
source install/local_setup.bash
```

Make a modification to a package (e.g. change the title in turtle_frame.cpp), rebuild, and run:
```bash
colcon build
ros2 run turtlesim turtlesim_node
```


## 5. macOS X11 forwarding setup (XQuartz)

Use this section before running GUI tools like turtlesim and rqt.

1. Install and open XQuartz.
2. In XQuartz preferences, open Security and enable Allow connections from network clients.
3. In macOS terminal, run:

```bash
defaults write org.xquartz.X11 nolisten_tcp -bool false
defaults read org.xquartz.X11
xhost +localhost
```

4. Optional quick validation:

```bash
docker run --rm -e DISPLAY=host.docker.internal:0 petedavidson887/xclock:0.0.1
```


## 6. ROS 2 main concepts and commands

### 6.1 ROS graph and discovery ("ROS Master" in ROS 1)

ROS 2 uses DDS (Data Distribution Service) for distributed discovery, so there is no central master process to start.

### 6.2 Nodes

Docs:
- https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Nodes/Understanding-ROS2-Nodes.html

Each node in ROS should be responsible for a single, modular purpose, such as controlling wheel motors or publishing data from a laser range-finder. Nodes can send and receive data through topics, services, actions, and parameters.

Run an executable from a package:

```bash
ros2 run <package_name> <executable_name>
```

Example:

```bash
ros2 run turtlesim turtlesim_node
```

Useful commands:

```bash
ros2 node list
ros2 node info <node_name>
```

#### 6.2.1 Remapping

Remapping allows you to reassign default node properties (node name, topic names, service names, etc.) to custom values.

```bash
ros2 run turtlesim turtlesim_node --ros-args -r __node:=my_turtle -r /turtle1/cmd_vel:=/cmd_vel
```

### 6.3 Topics

Topics are one of the main ways data moves between nodes and therefore between different parts of the system.

```bash
ros2 topic list -t
ros2 topic echo <topic_name>
ros2 interface show <topic_type>
ros2 topic info /turtle1/cmd_vel --verbose
ros2 topic find <topic_type>
```

General interface inspection (topics, services, actions):

```bash
ros2 interface show <type_name>
```

### 6.4 Services

Services are based on a call-and-response model, unlike the publisher-subscriber model used by topics.

```bash
ros2 service list -t
ros2 service type <service_name>
ros2 interface show <type_name>
ros2 service info <service_name>
ros2 service find <type_name>
ros2 service call /clear std_srvs/srv/Empty
```

### 6.5 Parameters

A parameter is a configuration value of a node. You can use integers, floats, booleans, strings, and lists.

```bash
ros2 param list
ros2 param get <node_name> <parameter_name>
ros2 param set <node_name> <parameter_name> <value>
ros2 param dump <node_name>
```

Useful examples:

```bash
ros2 param dump /turtlesim > turtlesim.yaml
ros2 param load /turtlesim turtlesim.yaml
ros2 run turtlesim turtlesim_node --ros-args --params-file turtlesim.yaml
```

### 6.6 Actions

Actions are intended for long-running tasks. They include a goal, feedback, and a result. Unlike services, actions can be canceled and provide continuous feedback.

```bash
ros2 action list -t
ros2 action type /turtle1/rotate_absolute
ros2 action info /turtle1/rotate_absolute
ros2 action send_goal /turtle1/rotate_absolute turtlesim/action/RotateAbsolute "{theta: 1.57}" --feedback
```


## 7. Tools and versions used

- macOS (host)
- XQuartz (X11 server on macOS)
- Docker Engine and Docker CLI
- ROS 2 Jazzy (inside container)
- ROS 2 CLI (ros2)
- DDS middleware (automatic discovery in ROS 2)
- rosdep (dependency resolution)
- colcon (workspace and package build tool)
- Optional GUI tools: turtlesim, rqt