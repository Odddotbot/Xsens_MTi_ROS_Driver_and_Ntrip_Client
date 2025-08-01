cmake_minimum_required(VERSION 3.5)
project(xsens_mti_ros2_driver)

if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 14)
endif()

# Find ROS dependencies
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(tf2 REQUIRED)
find_package(tf2_ros REQUIRED)
find_package(std_msgs REQUIRED)
find_package(geometry_msgs REQUIRED)
find_package(sensor_msgs REQUIRED)
find_package(nav_msgs REQUIRED)
find_package(nmea_msgs REQUIRED)
find_package(mavros_msgs REQUIRED)
find_package(diagnostic_msgs REQUIRED)
find_package(rosidl_default_generators REQUIRED)

# Detect ROS distro
if(DEFINED ENV{ROS_DISTRO})
  set(ROS_DISTRO $ENV{ROS_DISTRO})
else()
  set(ROS_DISTRO "unknown")
endif()

message(STATUS "Detected ROS_DISTRO: ${ROS_DISTRO}")

# Generate ROS interfaces
rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/XsStatusWord.msg"
  DEPENDENCIES std_msgs
)

ament_export_dependencies(rosidl_default_runtime)

# Include headers
include_directories(
  ${PROJECT_SOURCE_DIR}/lib/xspublic
  ${PROJECT_SOURCE_DIR}/include
  ${CMAKE_CURRENT_BINARY_DIR}/rosidl_generator_cpp
)

# Tell linker where to find the static libraries built by the Makefile
link_directories(
  ${CMAKE_CURRENT_SOURCE_DIR}/lib/xspublic/xscontroller
  ${CMAKE_CURRENT_SOURCE_DIR}/lib/xspublic/xscommon
  ${CMAKE_CURRENT_SOURCE_DIR}/lib/xspublic/xstypes
)

# Add custom target that builds all three libs by running the Makefile in lib/xspublic
add_custom_target(xspublic ALL
  COMMAND ${CMAKE_MAKE_PROGRAM} -C ${CMAKE_CURRENT_SOURCE_DIR}/lib/xspublic
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/lib/xspublic
  COMMENT "Building Xsens SDK libraries using Makefile"
)

# Create executable
add_executable(xsens_mti_node
  src/main.cpp
  src/xdainterface.cpp
  src/xdacallback.cpp
  src/xsens_log_handler.cpp
  src/ntrip_util.cpp
  src/xsens_time_handler.cpp
)

# Make sure the node depends on the Xsens SDK libs being built
add_dependencies(xsens_mti_node xspublic ${PROJECT_NAME})

# Link libraries (SDK libs + pthread + dl)
target_link_libraries(xsens_mti_node
  xscontroller
  xscommon
  xstypes
  pthread
  dl
)

# Link ROS dependencies
ament_target_dependencies(xsens_mti_node
  rclcpp tf2 tf2_ros std_msgs geometry_msgs sensor_msgs
  nav_msgs nmea_msgs mavros_msgs diagnostic_msgs rosidl_default_runtime
)

# ROS 2 typesupport handling for Foxy and later
if(${ROS_DISTRO} STREQUAL "foxy")
  target_link_libraries(xsens_mti_node
    ${CMAKE_BINARY_DIR}/libxsens_mti_ros2_driver__rosidl_typesupport_cpp.so
  )
  add_dependencies(xsens_mti_node
    ${PROJECT_NAME}__rosidl_typesupport_cpp
  )
  target_include_directories(xsens_mti_node PUBLIC
    ${CMAKE_CURRENT_BINARY_DIR}/rosidl_generator_cpp
  )
else()
  rosidl_get_typesupport_target(cpp_typesupport_target "${PROJECT_NAME}" "rosidl_typesupport_cpp")
  target_link_libraries(xsens_mti_node "${cpp_typesupport_target}")
endif()

# Installation rules
install(TARGETS xsens_mti_node
  EXPORT export_${PROJECT_NAME}
  DESTINATION lib/${PROJECT_NAME}
)

install(DIRECTORY launch param rviz urdf
  DESTINATION share/${PROJECT_NAME}
)

install(DIRECTORY headers/
  DESTINATION include/${PROJECT_NAME}
  FILES_MATCHING PATTERN "*.h"
)

ament_export_dependencies(
  rclcpp tf2 tf2_ros std_msgs geometry_msgs sensor_msgs nav_msgs
  nmea_msgs mavros_msgs diagnostic_msgs rosidl_default_runtime
)

ament_export_include_directories(include)

ament_package()
