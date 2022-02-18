# Setting up an Amazon picking challenge simulation with a UR10 robot arm in ROS / Gazebo.

## Step 1 - Install ROS, Gazebo and the Universal Robots packages
Install ROS (replace <distro> with required ROS distro) :
```
sudo apt update
sudo apt install ros-<distro>-desktop-full
```
Install Gazebo:
```
$ curl -sSL http://get.gazebosim.org | sh
```
If needed, make a catkin workspace in which your ROS system will sit (replace <distro> with ros distro) and source the setup file:
```
source /opt/ros/<distro>/setup.bash
mkdir -p ~/catkin_ws/src
catkin_make
source devel/setup.bash
```
Navigate to `/catkin_ws/src` and clone the universal_robotics GitHub:
```
$ git clone https://github.com/ros-industrial/universal_robot
```
Navigate back to `/catkin_ws` and build the workspace:
```
$ catkin_make
```

## Step 2 - Build the simulation
To launch the UR10 simulation (or other arm - just replace with the correct index), run the following:
```
roslaunch ur_gazebo ur_10_joint_limited.launch
```

Note that I have also made a change to the simulation environment and saved this within `myur10sim'. My moveit config file is also in there (see Step 3). For making basic changes to the simulation environment (URDF files), see this tutorial: https://www.youtube.com/watch?v=ayp87SjrwPc

## Step 3 - Setup MoveIt

To install MoveIt use the following with your ROS distro inserted:
```
$ sudo apt install ros-<distro>-moveit
```
Navigate to the `catkin_ws/src` folder (or a directory within that if preferred) and launch the moveit setup assistant:
```
$ roslaunch moveit_setup_assistant setup_assistant.launch
```

Follow the setup instructions in this tutorial: https://www.theconstructsim.com/control-gazebo-simulated-robot-moveit-video-answer/

You should now have the following files (see repo files to compare) in the newly setup MoveIt config folder.

```
controllers.yaml
joint_names.yaml
myur10_moveit_controller_manager.launch.xml
myur10_planning_execution.launch
```

To launch the RViz window which runs the planner, we first launch the Gazebo simulation to startup the ROS nodes and topics for our simulation, then we launch the MoveIt MotionPlanner in a new terminal window.
```
$ roslaunch myur10sim_description myur10.launch
$ roslaunch myur10_moveit_config myur10_planning_execution.launch
```

This launches both the Gazebo simulation and the RViz planning tool. To do a quick test that all is working, 'Add' the RobotModeln and 'Add' MotionPlanning to RViz. Set two states to move between (e.g. AllZeros to Home), hit Plan, and then hit Execute. When Executing, the robot in both RViz and Gazebo should move.

## Step 4 - Control the robot

So to recap, we have setup our catkin workspace in the ROS environment, we have installed the universal_robotics package in that workspace and we installed the ur10_moveit package and configured the config file using the setup assistant. We now want to program the UR10 to and for this we will use Python.

ROS works through a series of packages, which talk to one another through nodes and topics. The nodes are built in each individual package and the topics are the communication lines (serial comms) over which those nodes communicate.

The previously installed packages have a series of nodes which are used to control the robot, run Gazebo and more. We will now setup our own new package which will contain Python files that will communicate with our existing nodes to move the UR10.

Navigate back to the `/catkin_ws/src` folder and create a new package (see ROS Wiki 'Creating a Package' for more):
```
$ catkin_pkg_create ur10_python std_msgs rospy
$ cd ..
$ catkin_build
$ . devel/setup.bash
```

We now have a new package in our `src` folder that contains several files of its own - more on what these do available in the ROS wiki. We can now create a palce to save our Python scripts. Python with Moveit can be learnt about here: http://docs.ros.org/en/kinetic/api/moveit_tutorials/html/doc/move_group_python_interface/move_group_python_interface_tutorial.html

For now feel free to copy the scripts that are in this repo (heavily based on the tutorial) to see how this works. We also have to make them executable.
```
$ cd /ur10_python/src
$ mkdir scripts
$ gedit cartesian_controller.py
# Copy in python code
$ chmod a+x cartesian_controller.py
$ gedit master_controller.py
# Copy in python code
$ chmod a+x master_controller.py
```

Now because we've added two scripts, we need to update our package and make our catkin workspace again. Open the CMakeLists.txt file in the newly built ur10_python package and find the commented out lines that resemble the below code. Add in the two scripts we have built and then re-make the workspace:
```
$ gedit CMakeLists.txt
# In this file make sure to include:
catkin_install_python(PROGRAMS
  src/scripts/cartesian_controller.py
  src/scripts/master_controller.py
  DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)
```

These two Python scripts demonstrate how ROS nodes can be used to communicate with one another to control the UR10. The `cartesian_controller.py` script is used to setup the UR10 moveit controller with Python, it then moves the UR10 to a desired position in cartesian (x,y,z) coordinates.

The `master_controller.py` node publishes data, in this case just a randomised x-position, to the x_value topic. The `cartesian_controller.py` node subscribes to that topic in order to read x_values and move the UR10 to that position. This is  a constant stream of information that updates every 5 seconds, hence the robot should move (in Gazebo) every 5 seconds.So let's run it:
```
$ rosrun ur10_python cartesian_coordinates.py
$ rosrun ur10_python master_controller.py
```

Note that Gazebo and Rviz need to have been launched before running the python scripts.

















So we now want to control the robot using python node. To do so we setup a python launch file and a python ros node using rospy. This is done using the moveit package and based on the following documentation: http://docs.ros.org/en/kinetic/api/moveit_tutorials/html/doc/move_group_python_interface/move_group_python_interface_tutorial.html

Setup the python launch file by creating `myur10_move_group_python.launch` in the `myur10_moveit_config/launch` folder (see GitHub files for the content).

Setup the python file by creating a scripts folder under `myur10_moveit_config` within which the file `cartesian_controller.py` should be created (see GitHub files for the content). This can be used to control the robot in the cartesian plane. Create another file called 'master_controller.py' and again take the code from the github files. This one creates a node which publishes random x_coordinates for the robot to end effector to move to.

Now we can run that python file to control the robot (note there are some dependant python packages that may need installing - pip install.... etc).

In two seperate terminal windows run the following:

```
rosrun myur10_moveit_config master_controller.py

rosrun myur10_moveit_config_master_controller.py
```

Switching back to our Gazebo and RViz windows, we will now see the robot planning and then moving to the random positions. Note that if there is no possible path due to an dd randomised x_value, the movement will be aborted.

