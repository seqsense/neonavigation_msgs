cmake_minimum_required(VERSION 3.5...3.26)
project(safety_limiter_msgs)

# Interface files are auto-discovered instead of enumerated. The ROS1 (catkin)
# branch below consumes ${MSG_FILES}; the ROS2 branch lets
# rosidl_auto_generate_interfaces() do its own discovery.
file(GLOB MSG_FILES RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}/msg"
  CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/msg/*.msg")

if($ENV{ROS_VERSION} EQUAL 1)
  # === ROS1 ===
  find_package(catkin REQUIRED COMPONENTS
    message_generation
    sq_ros1_rclcpp_compat
    std_msgs
  )

  # The .msg sources use canonical ROS2 notation. SafetyLimiterStatus.msg
  # references builtin_interfaces/Time, so rewrite it to the ROS1 `time`
  # primitive before handing the copies to catkin's message generator.
  preprocess_ros1_interfaces(SUBDIR msg FILES ${MSG_FILES} OUTPUT_DIR_VAR ROS1_MSG_DIR)
  add_message_files(DIRECTORY ${ROS1_MSG_DIR} FILES ${MSG_FILES})
  generate_messages(DEPENDENCIES std_msgs)
  catkin_package(CATKIN_DEPENDS message_runtime std_msgs)

  # Emit ROS2-style compatibility headers (<pkg>/msg/*.hpp) into the devel
  # include space so hybrid consumers can include them on ROS1 as well.
  # Provided by sq_ros1_rclcpp_compat; must be done here, not in consumers.
  generate_ros1_compat_headers(
    PACKAGE "${PROJECT_NAME}"
    MSG_DIR "${CMAKE_CURRENT_SOURCE_DIR}/msg"
    OUTPUT_DIR "${CATKIN_DEVEL_PREFIX}/include")

elseif($ENV{ROS_VERSION} EQUAL 2)
  # === ROS2 ===
  find_package(ament_cmake_auto REQUIRED)
  ament_auto_find_build_dependencies()

  # Auto-discovers msg/ and resolves DEPENDENCIES from package.xml.
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
