cmake_minimum_required(VERSION 3.10.0)
project(neonavigation_msgs)

if($ENV{ROS_VERSION} EQUAL 1)
  find_package(catkin REQUIRED)
  catkin_metapackage()
endif()
