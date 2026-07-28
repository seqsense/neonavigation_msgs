cmake_minimum_required(VERSION 3.5)
project(neonavigation_metrics_msgs)

# helper.h uses a C++17 fold expression
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

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

  # The .msg sources use canonical ROS2 notation. Neither message references
  # builtin_interfaces/Time or /Duration, so they are handed to catkin's
  # generator as-is (no preprocess_ros1_interfaces() needed).
  add_message_files(FILES ${MSG_FILES})
  generate_messages(DEPENDENCIES std_msgs)
  catkin_package(
    INCLUDE_DIRS include
    CATKIN_DEPENDS message_runtime std_msgs
  )

  # Emit ROS2-style compatibility headers (<pkg>/msg/*.hpp) into the devel
  # include space so hybrid consumers - including helper.h in this package -
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
    catkin_add_gtest(test_helper test/src/test_helper.cpp)
    target_link_libraries(test_helper ${catkin_LIBRARIES})
    add_dependencies(test_helper ${${PROJECT_NAME}_EXPORTED_TARGETS})
  endif()

  install(DIRECTORY include/${PROJECT_NAME}/
    DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION}
  )
  # The installed helper.h includes <${PROJECT_NAME}/msg/*.hpp>, so the compat
  # headers generated above must reach the install space as well.
  install(DIRECTORY ${CATKIN_DEVEL_PREFIX}/include/${PROJECT_NAME}/msg/
    DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION}/msg
  )

elseif($ENV{ROS_VERSION} EQUAL 2)
  # === ROS2 ===
  find_package(ament_cmake_auto REQUIRED)
  ament_auto_find_build_dependencies()

  # Auto-discovers msg/ and resolves DEPENDENCIES from package.xml.
  rosidl_auto_generate_interfaces()

  if(BUILD_TESTING)
    find_package(ament_lint_auto REQUIRED)
    # uncrustify: the project formats C++ with clang-format instead.
    # copyright: the upstream BSD-3 header of this package (with the extra
    # "All rights reserved." line and "COPYRIGHT OWNER" wording) does not match
    # any ament_copyright license template, and the license text must not be
    # rewritten just to satisfy the linter.
    list(APPEND AMENT_LINT_AUTO_EXCLUDE
      ament_cmake_uncrustify
      ament_cmake_copyright)
    ament_lint_auto_find_test_dependencies()

    find_package(ament_cmake_gtest REQUIRED)
    ament_add_gtest(test_helper test/src/test_helper.cpp)
    target_include_directories(test_helper PRIVATE include)
    # Gives the test the generated message headers (and their build order).
    rosidl_get_typesupport_target(cpp_typesupport_target
      ${PROJECT_NAME} rosidl_typesupport_cpp)
    target_link_libraries(test_helper ${cpp_typesupport_target})
  endif()

  # Installs and exports include/ (header-only helper.h).
  ament_auto_package(USE_SCOPED_HEADER_INSTALL_DIR)

else()
  message(FATAL_ERROR "ROS_VERSION environment variable not set or unsupported.")
endif()
