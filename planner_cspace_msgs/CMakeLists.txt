cmake_minimum_required(VERSION 3.5)
project(planner_cspace_msgs)

# Interface files are auto-discovered instead of enumerated. The ROS1 (catkin)
# branch below consumes ${MSG_FILES} / ${ACTION_FILES}; the ROS2 branch lets
# rosidl_auto_generate_interfaces() do its own discovery.
file(GLOB MSG_FILES RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}/msg"
  CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/msg/*.msg")
file(GLOB ACTION_FILES RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}/action"
  CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/action/*.action")

if($ENV{ROS_VERSION} EQUAL 1)
  # === ROS1 ===
  find_package(catkin REQUIRED COMPONENTS
    actionlib_msgs
    geometry_msgs
    message_generation
    sq_ros1_rclcpp_compat
    std_msgs
  )

  # The interface sources use canonical ROS2 notation, but none of them
  # reference builtin_interfaces/Time or /Duration, so catkin can consume the
  # sources directly (no preprocess_ros1_interfaces() step needed).
  add_message_files(FILES ${MSG_FILES})
  add_action_files(DIRECTORY action FILES ${ACTION_FILES})
  generate_messages(DEPENDENCIES actionlib_msgs geometry_msgs std_msgs)
  catkin_package(CATKIN_DEPENDS message_runtime actionlib_msgs geometry_msgs std_msgs)

  # Emit ROS2-style compatibility headers (<pkg>/msg/*.hpp) into the devel
  # include space so hybrid consumers can include them on ROS1 as well.
  # Provided by sq_ros1_rclcpp_compat; must be done here, not in consumers.
  # Only .msg files get aliases; ROS1 actionlib and ROS2 rclcpp_action have
  # incompatible APIs, so action types are bridged in the interface layer.
  generate_ros1_compat_headers(
    PACKAGE "${PROJECT_NAME}"
    MSG_DIR "${CMAKE_CURRENT_SOURCE_DIR}/msg"
    OUTPUT_DIR "${CATKIN_DEVEL_PREFIX}/include")

elseif($ENV{ROS_VERSION} EQUAL 2)
  # === ROS2 ===
  find_package(ament_cmake_auto REQUIRED)
  ament_auto_find_build_dependencies()

  # Auto-discovers msg/ and action/ and resolves DEPENDENCIES from package.xml.
  rosidl_auto_generate_interfaces()

  if(BUILD_TESTING)
    find_package(ament_lint_auto REQUIRED)
    list(APPEND AMENT_LINT_AUTO_EXCLUDE ament_cmake_uncrustify)
    ament_lint_auto_find_test_dependencies()
  endif()

  ament_auto_package()

else()
  message(FATAL_ERROR "ROS_VERSION environment variable not set or unsupported.")
endif()
