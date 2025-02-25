# Aerial-Additive-Manufacturing


## How to install on Ubuntu 22.04:
Install ROS2 (https://docs.ros.org/en/humble/Installation.html)

Follow the steps from this https://github.com/TIERS/tello-ros2-gazebo and this https://github.com/ptrmu/fiducial_vlam repo to install the dependencies. When using ROS2 Humble, either clone the files from this repo in your workspace instead or make the necessary changes in the CMakeList.txt files.

Source your ros installation and colcon build.

## How to install using Docker (recommended)
Install Docker, Visual Studio Code and the Remote Development Extensions following this tutorial https://docs.ros.org/en/humble/How-To-Guides/Setup-ROS-2-with-VSCode-and-Docker-Container.html. Download the ws.zip folder in this repository and extract it. Click View->Command Palette...->Dev Containers: Open Folder in Container... in VSCode and select the ws folder. It will start building your Ubuntu 22.04 environment, ignore error messages as long as the container builds. To install the dependencies follow these steps:
```
source /opt/ros/humble/setup.bash
cd /home/ws 
sudo chown -R <USER> . 
colcon build 
source ./install/setup.bash 
sudo apt update 
rosdep update 
rosdep install --from-paths src --ignore-src -r -y 
sudo apt install libasio*
colcon build
source ./install/setup.bash
```
Make sure the Display variable in your Docker environment is correctly set, see the hint at the bottom of the Docker tutorial linked above.

## How to prepare your GCode and run the GCode to ROS nav_msgs/Path converter node:
Create a CAD file of the desired print geometry and save it as an STL file.
Slice the STL file with any slicing software (Tested with open source https://ultimaker.com/software/ultimaker-cura , used Marlin Flavour). You can create your own printer settings to have a larger print area or change parameters like layer height. Run `ros2 run gcode_to_path PathFromGcode --ros-args -p gcode_path:="<Path to your GCode>"` to publish the path and a boolean message if the drone should be currently extruding or not. 

## Starting the Simulation

`ros2 launch tello_gazebo vlam_launch.py`starts the gazebo simulation with three Tellos localizing themselves relative to fiducial markers. To make them takeoff `ros2 service call /drone1/tello_action tello_msgs/TelloAction "{cmd: 'takeoff'}"` has to be called (in total three times with the namespace changed to /drone2 and /drone3). 
The nodes can be run with:
`ros2 run print_controller PrintVisualization`
`ros2 run print_controller PrintController`
`ros2 run print_controller PayloadPositionPublisher`

Some topics maybe have to be remapped, depending on your usecase.

## Creating the map
'ros2 run print_controller MapGenerator' creates a map for the fiducial vlam with the marker with ID 0 starting at the top left and the offsets to the other markers specified in the programm. Currently it creates a map for 2x5 markers with the IDs from 1 to 10. Alternatively, you can print the A1 marker poster in this repository, which is already stored as a map.  

## Controlling the real drone
Connect to the Tello wifi, which appears after turning it on. Run 'ros2 launch print_controller start.py'. This launches the Tello Driver, Fiducial Vlam, and Print Controller. Make sure to change the map and GCode file paths in the launch file according to your environment.  Once the video stream of the tello is displayed run `ros2 service call /drone1/tello_action tello_msgs/TelloAction "{cmd: 'takeoff'}"` to make the Tello take off and follow the path. `ros2 service call /drone1/tello_action tello_msgs/TelloAction "{cmd: 'land'}"` makes it land again. 

## Changing hyperparameters
Hyperparameters like the p-value of the p-controller or the minimum distance to a point to consider it reached are defined at the very top of PrintControllerSingle.cpp. Creating a dynamic reconfigure node to change these values during run time is still work in progress.


## Starting Simulation 3 Drones
Before starting the Simulation, the models has to be imported
Open a command window and navigate to your workspace with cd ~/"workspacename"
Run the following commands:

    source install/setup.bash
    export GAZEBO_MODEL_PATH=${PWD}/install/tello_gazebo/share/tello_gazebo/models
    source /usr/share/gazebo/setup.sh
 
After that, you can start the simulation with 'ros2 launch print_controller simulate3drones_selfpositioning.py'
This starts the simulation of 3 drones. The drones take off and fly to their starting formation. When this positions is reached, you can send the path by running 'ros2 run gcode_to_path PathFromGcode --ros-args -p gcode_path:="<Path to your GCode>"'

The drones start to follow the path in their formation

## Starting 3 Drones in real life

To controll multiple drones with one PC/laptop etc. the wifi networks (and the sent udp packages) of the single drones have to be forwarded to one LAN-Network.
This was done according to https://github.com/clydemcqueen/flock2?tab=readme-ov-file and https://github.com/clydemcqueen/udp_forward.git


Write the image for the operating system of the Raspberry to an SD-card using a cardreader. (if you use an existing image and face problems with the size of the image, there are ways to shrink the image, you need to use an ubuntu operating system to do so)
You need one rasperry Pi for each drone. Once you wrote the image and started the RPi you also need to adapt the file 'telloX.sh', so the udp packages get forwarded to the correct IP adresses (based on your LAN-network IP adresses). There is an image in this repo which shows the networks structure and IP adresses i used.

Since we had problems with the RPi's onboard wifi module (driver crashed once videostream was started), we used extra USB wifidongles, one for each RPi.
There are probably ways to get rid of the RPi's and only use the USB wifidongles instead. (e.g. explained here https://medium.com/@henrymound/adventures-with-dji-ryze-tello-controlling-a-tello-swarm-1bce7d4e045d)

You need to setup the RPi's to automatically connect to one (and only one) of the drones, when the drones are powered on.
Once all 3 RPi's are powered on, you can power on the drones and the RPi's should connect to the wifi of the drones (one Raspberry per connects to one drone)

Then you can start the launchfile with 'ros2 launch print_controller 3drones_selfpositioning.py' which establishes the connection from ROS2 to the drones (uses the tello_driver_main node)
After 30 seconds the 3 drones automatically take off. I faced some problems with the take off:
  
  -> if the drone battery is low, they dont take off --> change battery
  
  -> rarely but sometimes one or more dont take off (reason is not known) --> if this happens just restart the launchfile

Then the drones fly to their starting positions in order to get to their flight formation.
Once they are in formation, send the path with 'ros2 run gcode_to_path PathFromGcode --ros-args -p gcode_path:="<Path to your GCode>"'
When they reached their start formation, the drones start to fly along the path maintaining their formation. 

To land the drones you need to call the following service:

'ros2 service call /drone1/tello_action tello_msgs/TelloAction "{cmd: 'land'}"'

'ros2 service call /drone2/tello_action tello_msgs/TelloAction "{cmd: 'land'}"'

'ros2 service call /drone3/tello_action tello_msgs/TelloAction "{cmd: 'land'}"'


## TIPS:

-try to avoid using a virtual machine,  this only rises problems with network connection, processing power etc --> set up a native Ubuntu 22.04 operating system (e.g dual boot) or use docker

-make sure, the poster with the markers is well illuminated, we used additional external light sources (with bad lighting you get a lot of encoder errors and this makes it unable to control the drones)

-look for alternatives to the Raspberrys and try to get a solution with only using the wifi dongles

-there are alternative ways to ROS2 to control the tello drones (e.g there is a python open source library, maybe have a look at the functionalities of that one too)




