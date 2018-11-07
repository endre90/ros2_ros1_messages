# ros2_ros1_messages

A ROS2 package providing an example and a guide on how to use the ROS2's package ros1_bridge with custom message types. Steps to add you own message types and succesfully bridge:

1. Build ROS2 from source: [https://index.ros.org/doc/ros2/Linux-Development-Setup/]
2. Define a message type in your ROS1 package (named for example: ros1_package) and save it in the msg folder of the package. Name the message, for example MessageX.msg.
3. Edit the CMakeLists.txt file of the ROS1 package:
    1. Uncomment the following section and add your message:
	```
	add_message_files(
  		FILES
		MessageX.msg
	)
	```
    2. Uncomment line 5 in the catkin_pakcage section:
	```
	catkin_package(
		#  INCLUDE_DIRS include
		#  LIBRARIES ros1_package
		#  CATKIN_DEPENDS other_catkin_pkg
		#  DEPENDS system_lib
    		CATKIN_DEPENDS message_runtime
	)
	```
    3. Uncomment the generate_messages section:
	```
	generate_messages(
		DEPENDENCIES
  		std_msgs  # Or other packages containing msgs
	)
	```
4. Edit the package.xml file so that you have the following lines uncommented:
```
<build_depend>message_generation</build_depend>
<exec_depend>message_runtime</exec_depend>
<buildtool_depend>catkin</buildtool_depend>
```
5. Source your ROS1 environment file, cd into your workspace and catkin_make.
6. Define a message type in the ros2_ros1_messages package (prefferably the same message type with the same name as in step 1) and save it in the msg folder of the ros2_ros1_messages package.
7. Define a mapping rule in a message_x_mapping_rules.yaml file between those two messages and save it in the ros2_ros1_messages package folder. The mapping rule file should have the following format (NOTE: two spaces define one indentation):
```
-
  ros1_package_name: 'ros1_pkg_name'
  ros1_message_name: 'ros1_msg_name'
  ros2_package_name: 'ros2_pkg_name'
  ros2_message_name: 'ros2_msg_name'
    fields_1_to_2:
    ros1_msg_field_1: 'ros2_msg_field_1'
    ros1_msg_field_2: 'ros2_msg_field_2'
    ros1_msg_field_3: 'ros2_msg_field_3'
```	
8. Edit the CMakeLists.txt file of the ROS2 ros2_ros1_messages package:
    1. Add your message type in the section:
	```
	rosidl_generate_interfaces(ros2_ros1_messages
  		"msg/Message1.msg"
  		"msg/Message2.msg"
  		"msg/MessageX.msg"
  		DEPENDENCIES builtin_interfaces
	)
	```
    2. After the BUILD_TESTING section, add the installation rule for the message_x_mapping_rules.yaml file:
	```
	install(
		FILES message_x_mapping_rules.yaml
  		DESTINATION share/${PROJECT_NAME})
	```
9. Edit the package.xml so that in the export section you add the mapping rule:
```
<export>
  <ros1_bridge mapping_rules="message_x_mapping_rules.yaml/">
  <build_type>ament_cmake</build_type>
</export>

```
10. You can use the ros1_bridge's dynamic_bridge, but for a setup where you have nonconsistent publishing and subscribing, it is more robust to have one persistent static bridge per topic. To do this, cd into the ros1_bridge's src folder and make a copy of the static_bridge.cpp and rename it as you like, for example: sb_topic_x.cpp. Open and edit:
```
// ROS 1 node
  ros::init(argc, argv, "ros1_sb_topic_x");
  ros::NodeHandle ros1_node;

  // ROS 2 node
  rclcpp::init(argc, argv);
  auto ros2_node = rclcpp::Node::make_shared("ros2_sb_topic_x");

  // bridge one example topic
  std::string topic_name = "topic_x";
  std::string ros1_type_name = "ros1_package/MessageX";
  std::string ros2_type_name = "ros2_ros1_bridge/MessageX";
  size_t queue_size = 10;
```
11. Open the CMakeLists.txt of the ros1_bridge package and edit it so that you add executable generation of the .cpp you have made in the previos step like:
```
custom_executable(sb_topic_x
  "src/sb_topic_x.cpp"
  ROS1_DEPENDENCIES
  TARGET_DEPENDENCIES ${ros2_message_packages})
target_link_libraries(sb_topic_x
  ${PROJECT_NAME})
```
12. Delete the /build/ros1_bridge and the /install/ros1_bridge folders (not exactly sure why this is needed, but the ros1_bridge's factory generator gets confused if the bridge is not built clean. Otherwise it will also compile, but some remains of the previous build can cause problems). Also, try not to name your ROS2 packages in a way so that they end either in "_msgs" or "_interfaces". Doing this might not cause problems, but it is better to avoid it since I have stumbled upon a few comments on this, saying that it might be probematic. 

13. In a fresh terminal, source your only your ROS2 environment, cd into the ROS2 workspace and build the workspace without the ros1_bridge with:
```
colcon build --symlink-install --packages-skip ros1_bridge
```
14. In another fresh terminal, source your ROS1 environment and then in the same terminal source the ROS2 environment. Now cd into your ROS2 workspace and build only the ros1_bridge with:
```
colcon build --symlink-install --packages-select ros1_bridge --cmake-force-configure
```
15. All done. You should be able to have your bridge up and running when you:
    1. Open a terminal and source the ROS1 environment
    2. Then, source the ROS2 environment
    3. ros2 run ros1_bridge sb_topic_x

