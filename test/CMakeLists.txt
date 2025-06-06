set(CMAKE_CXX_STANDARD 20)
if(NOT DEFINED CMAKE_BUILD_TYPE OR "${CMAKE_BUILD_TYPE}" STREQUAL "")
  set(CMAKE_BUILD_TYPE Release)
endif()

add_executable(forward_kinematics_test forward_kinematics_test.cpp)
add_executable(inverse_kinematics_test inverse_kinematics_test.cpp)

set(URDF_FILE ${CMAKE_SOURCE_DIR}/test/urdf/robot.urdf)
set(ROOT base_link)
set(TIP grasp_link)

# generate FK/IK implementation
set(FK_IMPL "FFK")
# set(FK_IMPL "FFK_CUDA") set(FK_IMPL "KDL")

if(${FK_IMPL} STREQUAL "FFK")
  target_compile_definitions(forward_kinematics_test PUBLIC USE_FAST_KINEMATICS)
  target_compile_definitions(inverse_kinematics_test PUBLIC USE_FAST_KINEMATICS)
  target_include_directories(forward_kinematics_test PUBLIC include)
  target_include_directories(inverse_kinematics_test PUBLIC include)
  generate_fast_forward_kinematics_library(
    fast_forward_kinematics_library
    URDF_FILE
    ${URDF_FILE}
    ROOT_LINK
    ${ROOT}
    TIP_LINK
    ${TIP})
  if(${CMAKE_BUILD_TYPE} STREQUAL "Release")
    set_target_properties(fast_forward_kinematics_library
                          PROPERTIES CMAKE_BUILD_TYPE Release)
    target_compile_options(fast_forward_kinematics_library PUBLIC -Ofast
                                                                  -march=native)
  endif()
  target_link_libraries(forward_kinematics_test
                        PUBLIC fast_forward_kinematics_library)
  target_compile_definitions(forward_kinematics_test PUBLIC MULTIPLIER=64)
  target_link_libraries(inverse_kinematics_test
                        PRIVATE fast_forward_kinematics_library)
  target_compile_definitions(inverse_kinematics_test PUBLIC MULTIPLIER=32)
elseif(${FK_IMPL} STREQUAL "FFK_CUDA")
  set(CMAKE_CUDA_ARCHITECTURES "native")
  target_compile_definitions(forward_kinematics_test PUBLIC USE_FAST_KINEMATICS)
  target_compile_definitions(inverse_kinematics_test PUBLIC USE_FAST_KINEMATICS)
  generate_fast_forward_kinematics_library_cuda(
    fast_forward_kinematics_library
    URDF_FILE
    ${URDF_FILE}
    ROOT_LINK
    ${ROOT}
    TIP_LINK
    ${TIP})
  if(${CMAKE_BUILD_TYPE} STREQUAL "Release")
    set_target_properties(fast_forward_kinematics_library
                          PROPERTIES CMAKE_BUILD_TYPE Release)
    target_compile_options(fast_forward_kinematics_library PUBLIC -O3)
  endif()
  target_link_libraries(forward_kinematics_test
                        PUBLIC fast_forward_kinematics_library)
  target_compile_definitions(forward_kinematics_test PUBLIC MULTIPLIER=64)
  target_link_libraries(inverse_kinematics_test
                        PRIVATE fast_forward_kinematics_library)
  target_compile_definitions(inverse_kinematics_test PUBLIC MULTIPLIER=32)
else()
  set(CXX_FLAGS "-Ofast")

  set(LIB_KDL_PARSER
      ${CMAKE_CURRENT_BINARY_DIR}/kdl_parser/src/kdl-parser-build/libkdl_parser.so
  )
  include(ExternalProject)

  ExternalProject_Add(
    kdl-parser
    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/kdl_parser
    SOURCE_SUBDIR kdl_parser
    GIT_REPOSITORY https://github.com/ros/kdl_parser.git
    GIT_TAG 2.12.0
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR}
               -DCMAKE_CXX_FLAGS=${CXX_FLAGS}
    BUILD_BYPRODUCTS ${LIB_KDL_PARSER})
  # use custom target to ensure kdl-parser is downloaded and installed before
  # building main target
  add_library(kdl_parser_lib SHARED IMPORTED)
  add_custom_target(kdl_parser_lib-target DEPENDS kdl-parser)
  add_dependencies(kdl_parser_lib kdl_parser_lib-target)
  set_target_properties(kdl_parser_lib PROPERTIES IMPORTED_LOCATION
                                                  ${LIB_KDL_PARSER})

  set(LIB_KDL
      ${CMAKE_CURRENT_BINARY_DIR}/kdl/src/kdl-content-build/src/liborocos-kdl.so
  )
  ExternalProject_Add(
    kdl-content
    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/kdl
    SOURCE_SUBDIR orocos_kdl
    GIT_REPOSITORY https://github.com/orocos/orocos_kinematics_dynamics.git
    GIT_TAG master # v1.5.1
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${CMAKE_BINARY_DIR}
               -DCMAKE_CXX_FLAGS=${CXX_FLAGS}
    BUILD_BYPRODUCTS ${LIB_KDL})
  # use custom target to ensure kdl-content is downloaded and installed before
  # building main target
  add_library(kdl_lib SHARED IMPORTED)
  add_custom_target(kdl_lib-target DEPENDS kdl-content)
  add_dependencies(kdl_lib kdl_lib-target)
  set_target_properties(kdl_lib PROPERTIES IMPORTED_LOCATION ${LIB_KDL})

  find_package(Eigen3 3.3 NO_MODULE)
  add_library(kdl_implementation SHARED kdl/kdl_implementation.cpp)
  target_include_directories(kdl_implementation PUBLIC ../include)
  target_include_directories(kdl_implementation PUBLIC include)
  target_compile_definitions(kdl_implementation PUBLIC NUMBER_OF_JOINTS=6)
  target_compile_definitions(kdl_implementation PUBLIC URDF_FILE=${URDF_FILE})
  target_compile_definitions(kdl_implementation PUBLIC ROOT=${ROOT})
  target_compile_definitions(kdl_implementation PUBLIC TIP=${TIP})
  if(${CMAKE_BUILD_TYPE} STREQUAL "Release")
    set_target_properties(kdl_implementation PROPERTIES CMAKE_BUILD_TYPE
                                                        Release)
    target_compile_options(kdl_implementation PUBLIC ${CXX_FLAGS_ARGS})
  endif()
  target_link_libraries(kdl_implementation PUBLIC kdl_lib kdl_parser_lib
                                                  Eigen3::Eigen)

  target_include_directories(kdl_implementation
                             PUBLIC ${CMAKE_BINARY_DIR}/include/kdl)
  target_include_directories(kdl_implementation
                             PUBLIC ${CMAKE_BINARY_DIR}/include/kdl_parser)

  target_link_libraries(forward_kinematics_test PUBLIC kdl_implementation)
  target_link_libraries(inverse_kinematics_test PUBLIC kdl_implementation)

  target_compile_definitions(forward_kinematics_test PUBLIC MULTIPLIER=2)
  target_compile_definitions(inverse_kinematics_test PUBLIC MULTIPLIER=1)

endif()
