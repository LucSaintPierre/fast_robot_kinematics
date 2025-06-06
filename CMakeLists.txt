cmake_minimum_required(VERSION 3.22)
project(fast_forward_kinematics)

set(FFK_VERSION_MAJOR 0)
set(FFK_VERSION_MINOR 1)
set(FFK_VERSION ${FFK_VERSION_MAJOR}.${FFK_VERSION_MINOR})

option(BUILD_TESTING "Build fast_forward_kinematics tests" OFF)

# Include GNUInstallDirs to get canonical paths
include(GNUInstallDirs)
include(CTest)

# Add cmake macro to project
include(fast_forward_kinematics.cmake)

add_library(fast_forward_kinematics_header INTERFACE)
target_include_directories(
  fast_forward_kinematics_header
  INTERFACE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
            $<INSTALL_INTERFACE:include/fast_forward_kinematics>)
if(BUILD_TESTING)
  add_subdirectory(test)
endif()

include(CMakePackageConfigHelpers)

configure_package_config_file(
  fast_forward_kinematics-config.cmake.in fast_forward_kinematics-config.cmake
  INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/fast_forward_kinematics"
  PATH_VARS CMAKE_INSTALL_INCLUDEDIR CMAKE_INSTALL_LIBDIR
  NO_CHECK_REQUIRED_COMPONENTS_MACRO)

write_basic_package_version_file(
  fast_forward_kinematics-config-version.cmake
  VERSION ${FFK_VERSION}
  COMPATIBILITY AnyNewerVersion)

install(
  FILES "${CMAKE_BINARY_DIR}/fast_forward_kinematics-config.cmake"
        "${CMAKE_BINARY_DIR}/fast_forward_kinematics-config-version.cmake"
        "${CMAKE_SOURCE_DIR}/fast_forward_kinematics.cmake"
  DESTINATION "${CMAKE_INSTALL_DATADIR}/fast_forward_kinematics/cmake")

install(DIRECTORY "${CMAKE_SOURCE_DIR}/code_generation"
        DESTINATION "${CMAKE_INSTALL_DATADIR}/fast_forward_kinematics/cmake")
install(DIRECTORY include/
        DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/fast_forward_kinematics")

# Create an export set
install(TARGETS fast_forward_kinematics_header
        EXPORT fast_forward_kinematicsTargets)
# Targets files
export(EXPORT fast_forward_kinematicsTargets
       FILE ${CMAKE_CURRENT_BINARY_DIR}/fast_forward_kinematics-targets.cmake)
install(
  EXPORT fast_forward_kinematicsTargets
  FILE fast_forward_kinematics-targets.cmake
  DESTINATION "${CMAKE_INSTALL_DATADIR}/fast_forward_kinematics/cmake")
