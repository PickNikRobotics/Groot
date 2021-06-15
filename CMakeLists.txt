cmake_minimum_required(VERSION 3.2)

# Update the policy setting to avoid an error when loading the ament_cmake package
# at the current cmake version level
if(POLICY CMP0057)
    cmake_policy(SET CMP0057 NEW)
endif()

project(groot)

find_package(NodeEditor COMPONENTS nodes REQUIRED)
find_package(Qt5 COMPONENTS  Core Widgets Gui OpenGL Xml Svg REQUIRED)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH}  "${CMAKE_CURRENT_LIST_DIR}/cmake")

if(NOT CMAKE_VERSION VERSION_LESS 3.1)
    set(CMAKE_CXX_STANDARD 11)
    set(CMAKE_CXX_STANDARD_REQUIRED ON)
else()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
endif()

#############################################################
find_package(ament_cmake QUIET)
if(ament_cmake_FOUND)
  find_package(behaviortree_cpp_v3 REQUIRED)

  set(dependencies
    behaviortree_cpp_v3
    NodeEditor
  )

  message(STATUS "------------------------------------------------")
  message(STATUS "BehaviourTreeEditor is being built using AMENT.")
  message(STATUS "------------------------------------------------")
elseif( CATKIN_DEVEL_PREFIX OR CATKIN_BUILD_BINARY_PACKAGE)
# http://answers.ros.org/question/230877/optionally-build-a-package-with-catkin/
  set(catkin_FOUND 1)
  # add_definitions( -DUSING_ROS )

  find_package(catkin REQUIRED COMPONENTS behaviortree_cpp_v3)

  message(STATUS "------------------------------------------------")
  message(STATUS "BehaviourTreeEditor is being built using CATKIN.")
  message(STATUS "------------------------------------------------")

  catkin_package(
    INCLUDE_DIRS 
    LIBRARIES qt_node_editor
    CATKIN_DEPENDS behaviortree_cpp_v3
    DEPENDS 
    )
endif()
#############################################################

set(CMAKE_INCLUDE_CURRENT_DIR ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTOUIC ON)
set(CMAKE_AUTORCC ON)

add_definitions(${Qt5Widgets_DEFINITIONS} -DNODE_EDITOR_SHARED )
set(CMAKE_CXX_FLAGS "${Qt5Widgets_EXECUTABLE_COMPILE_FLAGS}")

#set (CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -fno-omit-frame-pointer -fsanitize=address")
#set (CMAKE_LINKER_FLAGS_DEBUG "${CMAKE_STATIC_LINKER_FLAGS_DEBUG} -fno-omit-frame-pointer -fsanitize=address")

if(ament_cmake_FOUND)
  INCLUDE_DIRECTORIES(include ./ ./bt_editor/)
else()
  INCLUDE_DIRECTORIES(${catkin_INCLUDE_DIRS} ./QtNodeEditor/include ./  )
endif()

# add_subdirectory( QtNodeEditor )

##########################################
# FIX ME in the future....

set(BUILD_EXAMPLES   OFF CACHE BOOL "")
set(BUILD_UNIT_TESTS OFF CACHE BOOL "")
set(BUILD_TOOLS OFF CACHE BOOL "")
set(BUILD_TESTING OFF CACHE BOOL "")
set(CATKIN_ENABLE_TESTING OFF CACHE BOOL "")

if(NOT catkin_FOUND AND NOT ament_cmake_FOUND)
    # look for BehaviorTree.CPP on the system
    find_package(BehaviorTreeV3)
    if (NOT BehaviorTree_FOUND)
        # use git submodule only if you are not compiling with catkin
        add_subdirectory( depend/BehaviorTree.CPP )
        include_directories( depend/BehaviorTree.CPP/include )
    endif()
endif()

##########################################

set(APP_CPPS
    ./bt_editor/models/BehaviorTreeNodeModel.cpp
    ./bt_editor/models/SubtreeNodeModel.cpp

    ./bt_editor/mainwindow.cpp
    ./bt_editor/editor_flowscene.cpp
    ./bt_editor/utils.cpp
    ./bt_editor/bt_editor_base.cpp
    ./bt_editor/graphic_container.cpp
    ./bt_editor/startup_dialog.cpp

    ./bt_editor/sidepanel_editor.cpp
    ./bt_editor/sidepanel_replay.cpp
    ./bt_editor/custom_node_dialog.cpp

    ./bt_editor/XML_utilities.cpp
    )

set(RESOURCE_FILES
    ./bt_editor/resources/icons.qrc
    ./bt_editor/resources/style.qrc
    ./test_data/test_files.qrc
    )

set(FORMS_UI
  ./bt_editor/about_dialog.ui
  ./bt_editor/mainwindow.ui
  ./bt_editor/action_form.ui
  ./bt_editor/sidepanel_editor.ui
  ./bt_editor/sidepanel_replay.ui
  ./bt_editor/startup_dialog.ui
  ./bt_editor/custom_node_dialog.ui
  )

find_package(ZMQ)

if( ZMQ_FOUND )
    message(STATUS "ZeroMQ found.")
    add_definitions( -DZMQ_FOUND )

    set(APP_CPPS ${APP_CPPS} ./bt_editor/sidepanel_monitor.cpp )
    set(FORMS_UI ${FORMS_UI} ./bt_editor/sidepanel_monitor.ui )

else()
    message(WARNING "ZeroMQ NOT found. Real-time monitoring disabled.")
endif()

QT5_WRAP_UI(FORMS_HEADERS ${FORMS_UI})

add_library(behavior_tree_editor SHARED
    ${APP_CPPS}
    ${FORMS_HEADERS}
)

SET(GROOT_DEPENDENCIES NodeEditor)

if(ament_cmake_FOUND)
    ament_target_dependencies(behavior_tree_editor ${dependencies})
elseif( catkin_FOUND )
    SET(GROOT_DEPENDENCIES ${GROOT_DEPENDENCIES} ${catkin_LIBRARIES} )
else()
    SET(GROOT_DEPENDENCIES ${GROOT_DEPENDENCIES} behaviortree_cpp_v3 )
endif()

if( ZMQ_FOUND )
    SET(GROOT_DEPENDENCIES ${GROOT_DEPENDENCIES} ZMQ)
endif()

# target_link_libraries(behavior_tree_editor
#   INTERFACE NodeEditor::nodes
#   PUBLIC behavior_tree_editor )


add_executable(Groot ./bt_editor/main.cpp  ${RESOURCE_FILES})
target_link_libraries(Groot
  PUBLIC NodeEditor::nodes
  PUBLIC behavior_tree_editor)

add_subdirectory(test)

######################################################
# INSTALL
if(ament_cmake_FOUND)
    set( GROOT_LIB_DESTINATION   lib )
    set( GROOT_INC_DESTINATION   include )
    set( GROOT_BIN_DESTINATION   lib/${PROJECT_NAME} )
elseif(catkin_FOUND)
    set( GROOT_LIB_DESTINATION   ${CATKIN_PACKAGE_LIB_DESTINATION} )
    set( GROOT_INC_DESTINATION   ${CATKIN_PACKAGE_INCLUDE_DESTINATION} )
    set( GROOT_BIN_DESTINATION   ${CATKIN_PACKAGE_BIN_DESTINATION} )
else()
    set( GROOT_LIB_DESTINATION   lib )
    set( GROOT_INC_DESTINATION   include )
    set( GROOT_BIN_DESTINATION   bin )
    
    INSTALL( FILES ${CMAKE_CURRENT_SOURCE_DIR}/groot_icon.png
			 DESTINATION share/icons/hicolor/256x256)
			 
	INSTALL( FILES ${CMAKE_CURRENT_SOURCE_DIR}/Groot.desktop
			 DESTINATION share/applications)
    
endif()

INSTALL(TARGETS behavior_tree_editor
  LIBRARY DESTINATION ${GROOT_LIB_DESTINATION}
  ARCHIVE DESTINATION lib
  INCLUDES DESTINATION include
)

# configure_package_config_file(${CMAKE_CURRENT_LIST_DIR}/cmake/.cmake.in
#   ${CMAKE_CURRENT_BINARY_DIR}/NodeEditorConfig.cmake
#   INSTALL_DESTINATION ${INSTALL_CONFIGDIR}
# )

INSTALL(FILES cmake/FindZMQ.cmake DESTINATION lib/cmake)

INSTALL(TARGETS Groot RUNTIME DESTINATION ${GROOT_BIN_DESTINATION} )

# this didn't work:
# INSTALL(TARGETS Groot INCLUDES destination include)
INSTALL(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/bt_editor/ DESTINATION ${GROOT_INC_DESTINATION})

# this installs the QtNodeEditor headers to the groot package, but when trying to build moveit_studio_manual_mode_ui
# with a groot header included, colcon complains about not finding cmake files for QtNodeEditor:
#
# CMake Error at /home/wyatt/ws_moveit2/install/groot/share/groot/cmake/ament_cmake_export_dependencies-extras.cmake:21 (find_package):
#   By not providing "FindQtNodeEditor.cmake" in CMAKE_MODULE_PATH this project
#   has asked CMake to find a package configuration file provided by
#   "QtNodeEditor", but CMake did not find one.

#   Could not find a package configuration file provided by "QtNodeEditor" with
#   any of the following names:

#     QtNodeEditorConfig.cmake
#     qtnodeeditor-config.cmake

#   Add the installation prefix of "QtNodeEditor" to CMAKE_PREFIX_PATH or set
#   "QtNodeEditor_DIR" to a directory containing one of the above files.  If
#   "QtNodeEditor" provides a separate development package or SDK, be sure it
#   has been installed.
# Call Stack (most recent call first):
#   /home/wyatt/ws_moveit2/install/groot/share/groot/cmake/grootConfig.cmake:41 (include)
#   CMakeLists.txt:36 (find_package)

# INSTALL(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/QtNodeEditor/include/ DESTINATION ${GROOT_INC_DESTINATION})

if(ament_cmake_FOUND)
  ament_export_include_directories(include)
  ament_export_dependencies(${GROOT_DEPENDENCIES} ${dependencies})
  ament_export_libraries(behavior_tree_editor)
  ament_package()
endif()
