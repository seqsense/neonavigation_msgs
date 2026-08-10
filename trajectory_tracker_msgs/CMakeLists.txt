cmake_minimum_required(VERSION 3.5...3.26)
project(trajectory_tracker_msgs)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Interface files are auto-discovered instead of enumerated. The ROS1 (catkin)
# branch below consumes ${MSG_FILES} / ${SRV_FILES}; the ROS2 branch lets
# rosidl_auto_generate_interfaces() do its own discovery.
file(GLOB MSG_FILES RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}/msg"
  CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/msg/*.msg")
file(GLOB SRV_FILES RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}/srv"
  CONFIGURE_DEPENDS "${CMAKE_CURRENT_SOURCE_DIR}/srv/*.srv")

if($ENV{ROS_VERSION} EQUAL 1)
  # === ROS1 ===
  # nav_msgs is a real dependency of converter.h (not only of the test), and
  # sq_ros1_rclcpp_compat supplies the ROS2-style compat headers for the
  # geometry_msgs/nav_msgs types that converter.h spells as <pkg>::msg::<Type>.
  find_package(catkin REQUIRED COMPONENTS
    geometry_msgs
    message_generation
    nav_msgs
    sq_ros1_rclcpp_compat
    std_msgs
  )

  # The interface sources use canonical ROS2 notation. None of them reference
  # builtin_interfaces/Time or /Duration, so they are handed to catkin's
  # generator as-is (no preprocess_ros1_interfaces() needed).
  add_message_files(FILES ${MSG_FILES})
  add_service_files(FILES ${SRV_FILES})
  generate_messages(DEPENDENCIES geometry_msgs std_msgs)
  catkin_package(
    INCLUDE_DIRS include
    CATKIN_DEPENDS geometry_msgs message_runtime nav_msgs sq_ros1_rclcpp_compat std_msgs
  )

  # Emit ROS2-style compatibility headers (<pkg>/msg/*.hpp) into the devel
  # include space so hybrid consumers - including converter.h in this package -
  # can include them on ROS1 as well. Provided by sq_ros1_rclcpp_compat;
  # must be done here, not in consumers.
  generate_ros1_compat_headers(
    PACKAGE "${PROJECT_NAME}"
    MSG_DIR "${CMAKE_CURRENT_SOURCE_DIR}/msg"
    OUTPUT_DIR "${CATKIN_DEVEL_PREFIX}/include")

  include_directories(include ${catkin_INCLUDE_DIRS})

  if(CATKIN_ENABLE_TESTING)
    # Style checks are unified on the ROS2 side (ament_lint_auto); roslint is
    # intentionally not used any more.
    catkin_add_gtest(test_path_with_velocity_conversion
      test/src/test_path_with_velocity_conversion.cpp)
    target_link_libraries(test_path_with_velocity_conversion ${catkin_LIBRARIES})
    add_dependencies(test_path_with_velocity_conversion ${${PROJECT_NAME}_EXPORTED_TARGETS})
  endif()

  install(DIRECTORY include/${PROJECT_NAME}/
    DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION}
  )
  # The installed converter.h includes <${PROJECT_NAME}/msg/*.hpp>, so the
  # compat headers generated above must reach the install space as well.
  install(DIRECTORY ${CATKIN_DEVEL_PREFIX}/include/${PROJECT_NAME}/msg/
    DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION}/msg
  )

elseif($ENV{ROS_VERSION} EQUAL 2)
  # === ROS2 ===
  find_package(ament_cmake_auto REQUIRED)
  ament_auto_find_build_dependencies()

  # Auto-discovers msg/ and srv/ and resolves DEPENDENCIES from package.xml.
  rosidl_auto_generate_interfaces()

  if(BUILD_TESTING)
    find_package(ament_lint_auto REQUIRED)
    # uncrustify: the project formats C++ with clang-format instead.
    # copyright: the upstream BSD-3 header of this package (with the extra
    # "All rights reserved." line and "COPYRIGHT OWNER" wording) matches no
    # ament_copyright license template - it is reported as license=<unknown> -
    # and the license text must not be rewritten just to satisfy the linter.
    list(APPEND AMENT_LINT_AUTO_EXCLUDE
      ament_cmake_uncrustify
      ament_cmake_copyright)
    ament_lint_auto_find_test_dependencies()

    find_package(ament_cmake_gtest REQUIRED)
    ament_add_gtest(test_path_with_velocity_conversion
      test/src/test_path_with_velocity_conversion.cpp)
    target_include_directories(test_path_with_velocity_conversion PRIVATE include)
    ament_target_dependencies(test_path_with_velocity_conversion
      geometry_msgs nav_msgs std_msgs)
    # Gives the test the generated message headers (and their build order).
    rosidl_get_typesupport_target(cpp_typesupport_target
      ${PROJECT_NAME} rosidl_typesupport_cpp)
    target_link_libraries(test_path_with_velocity_conversion ${cpp_typesupport_target})
  endif()

  # Installs and exports include/ (header-only converter.h).
  ament_auto_package(USE_SCOPED_HEADER_INSTALL_DIR)

else()
  message(FATAL_ERROR "ROS_VERSION environment variable not set or unsupported.")
endif()
