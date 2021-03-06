project(opentrack)
cmake_minimum_required(VERSION 2.8.11)
cmake_policy(SET CMP0020 NEW)

include(CMakeParseArguments)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_SOURCE_DIR}/cmake/)
include(GetGitRevisionDescription)
include(FindPkgConfig)
find_package(Git QUIET)
if(GIT_FOUND)
    git_describe(OPENTRACK__COMMIT --tags --always)
endif()

SET(CMAKE_BUILD_WITH_INSTALL_RPATH TRUE)
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
SET(CMAKE_SKIP_INSTALL_RPATH FALSE)
SET(CMAKE_SKIP_RPATH FALSE)
SET(CMAKE_INSTALL_RPATH ${CMAKE_INSTALL_PREFIX})
set(CMAKE_INCLUDE_CURRENT_DIR ON)
set(CMAKE_AUTOMOC OFF)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

find_package(OpenCV)
include_directories(${OpenCV_INCLUDE_DIRS})

find_package(Qt5 REQUIRED COMPONENTS Core Xml Network Widgets Gui ${maybe-serial-port} QUIET)
include_directories(${Qt5Core_INCLUDE_DIRS} ${Qt5Xml_INCLUDE_DIRS} ${Qt5Gui_INCLUDE_DIRS} ${Qt5Widgets_INCLUDE_DIRS} ${Qt5Network_INCLUDE_DIRS})
add_definitions(${Qt5Core_DEFINITIONS} ${Qt5Xml_DEFINITIONS} ${Qt5Gui_DEFINITIONS} ${Qt5Widgets_DEFINITIONS} ${Qt5Network_DEFINITIONS})

set(my-qt-deps)
if(WIN32) # hack to avoid breakage on buildbot
    set(my-qt-deps ws2_32)
endif()
set(MY_QT_LIBS ${Qt5Widgets_LIBRARIES} ${Qt5Gui_LIBRARIES} ${Qt5Network_LIBRARIES} ${Qt5Xml_LIBRARIES} ${Qt5Core_LIBRARIES} ${my-qt-deps})

# note, hatire supports both ftnoir and opentrack
# don't remove without being sure as hell -sh 20140922
add_definitions(-DOPENTRACK_API)

if(${CMAKE_CXX_COMPILER_ID} STREQUAL "Clang")
    set(CMAKE_COMPILER_IS_GNUCC TRUE)
    set(CMAKE_COMPILER_IS_GNUCXX TRUE)
endif()

if(CMAKE_COMPILER_IS_GNUCXX AND NOT APPLE)
    if(MINGW)
        set(version-script mingw)
    else()
        set(version-script posix)
    endif()
endif()

if(APPLE)
    set(apple-frameworks "-stdlib=libc++ -framework Cocoa -framework CoreFoundation -lobjc -lz -framework Carbon")
    set(CMAKE_SHARED_LINKER_FLAGS " ${apple-frameworks} ${CMAKE_SHARED_LINKER_FLAGS}")
    set(CMAKE_STATIC_LINKER_FLAGS " ${apple-frameworks} ${CMAKE_STATIC_LINKER_FLAGS}")
    set(CMAKE_EXE_LINKER_FLAGS " ${apple-frameworks} ${CMAKE_EXE_LINKER_FLAGS}")
    set(CMAKE_MODULE_LINKER_FLAGS " ${apple-frameworks} ${CMAKE_MODULE_LINKER_FLAGS}")
endif()

if(CMAKE_COMPILER_IS_GNUCXX OR APPLE)
    set(CMAKE_CXX_FLAGS " -std=c++11 ${CMAKE_CXX_FLAGS} ")
endif()

# qt broken as usual
set(EXTRA-MOCS opentrack/options.hpp)

macro(opentrack_module n dir)
    file(GLOB ${n}-c ${dir}/*.cpp ${dir}/*.c ${dir}/*.h ${dir}/*.hpp ${EXTRA-MOCS})
    file(GLOB ${n}-res ${dir}/*.rc)
    file(GLOB ${n}-ui ${dir}/*.ui)
    file(GLOB ${n}-rc ${dir}/*.qrc)
endmacro()

macro(opentrack_qt n)
    qt5_wrap_cpp(${n}-moc ${${n}-c} OPTIONS --no-notes)
    QT5_WRAP_UI(${n}-uih ${${n}-ui})
    QT5_ADD_RESOURCES(${n}-rcc ${${n}-rc})
    set(${n}-all ${${n}-c} ${${n}-rc} ${${n}-rcc} ${${n}-uih} ${${n}-moc} ${${n}-res})
endmacro()

macro(opentrack_library n dir)
    cmake_parse_arguments(foolib "" "LINK;COMPILE" "" ${ARGN})
    if(NOT " ${foolib_UNPARSED_ARGUMENTS}" STREQUAL " ")
        message(FATAL_ERROR "opentrack_library bad formals")
    endif()
    opentrack_module(${n} ${dir})
    opentrack_qt(${n})
    add_library(${n} SHARED ${${n}-all})
    target_link_libraries(${n} ${MY_QT_LIBS})
    if(CMAKE_COMPILER_IS_GNUCXX AND NOT APPLE)
        SET_TARGET_PROPERTIES(${n} PROPERTIES
            LINK_FLAGS "${foolib_LINK} -Wl,--version-script=${CMAKE_SOURCE_DIR}/opentrack/${version-script}-version-script.txt"
            COMPILE_FLAGS "${foolib_COMPILE} -fvisibility=hidden -fvisibility-inlines-hidden"
        )
    else()
        set_target_properties(${n} PROPERTIES LINK_FLAGS ${foolib_LINK} COMPILE_FLAGS ${foolib_COMPILE})
    endif()
    install(TARGETS ${n} RUNTIME DESTINATION . LIBRARY DESTINATION .)
endmacro()

function(link_with_dinput8 n)
    if(WIN32)
        target_link_libraries(${n} dinput8 dxguid strmiids)
    endif()
endfunction()

# ----

# cache variables

# ----

IF(WIN32)
    SET(SDK_VJOY "" CACHE PATH "VJoy SDK path")
    SET(SDK_CONSOLE_DEBUG FALSE CACHE BOOL "Console window visible at runtime")
ENDIF()

IF("${CMAKE_SYSTEM}" MATCHES "Linux" OR APPLE)
    set(SDK_XPLANE "" CACHE PATH "Path to X-Plane SDK")
    set(SDK_ENABLE_LIBEVDEV FALSE CACHE BOOL "libevdev virtual joystick protocol support")
endif()

SET(SDK_HYDRA "" CACHE PATH "libSixense path for Razer Hydra")
SET(SDK_HYDRA_AMD64 FALSE CACHE BOOL "whether libSixense is amd64 (else ia-32)")
SET(SDK_RIFT "" CACHE PATH "libOVR path for Oculus Rift")

set(SDK_ARUCO_LIBPATH "" CACHE FILEPATH "Aruco paper marker tracker static library path")

if(WIN32)
    set(SDK_SIMCONNECT "" CACHE PATH "SimConnect SDK path for MS FSX")
    set(SDK_FSUIPC "" CACHE PATH "FSUIPC for older MS FSX path")
endif()

if(NOT WIN32)
    set(SDK_WINE_PREFIX "" CACHE PATH "Wine install prefix")
    set(SDK_WINE_NO_WRAPPER FALSE CACHE BOOL "disable Wine wrapper -- use Wine only for X-Plane")
endif()

# ----

# misc

# ----

string(TIMESTAMP filename-date "%Y%m%d")
set(filename-ostype ${CMAKE_SYSTEM_NAME})
get_git_head_revision(filename-branch_0 filename-hash_0)
if(filename-hash_0)
    string(SUBSTRING "${filename-hash_0}" 0 7 filename-hash)
endif()
string(REPLACE "refs/heads/" "" filename-branch_1 "${filename-branch_0}")
string(REPLACE "/" "-" filename-branch "${filename-branch_1}")
set(filename_0 opentrack-${filename-ostype}-${filename-branch}-${filename-date}-${filename-hash})
string(TOLOWER "${filename_0}" filename_1)
set(filename "${CMAKE_BINARY_DIR}/${filename_1}.zip")

add_custom_command(OUTPUT ${filename} COMMAND env sh "${CMAKE_SOURCE_DIR}/make-tar.sh" "${CMAKE_INSTALL_PREFIX}" "${filename}")
add_custom_target(tarball DEPENDS ${filename})


opentrack_module(opentrack-api opentrack)
opentrack_qt(opentrack-api)
add_library(opentrack-api STATIC ${opentrack-api-all})
target_link_libraries(opentrack-api ${MY_QT_LIBS})

if(NOT WIN32)
    target_link_libraries(opentrack-api dl)
else()
    target_link_libraries(opentrack-api winmm)
endif()

# ----

# conditional targets

# ----

if(SDK_XPLANE)
    INCLUDE_DIRECTORIES(${SDK_XPLANE}/CHeaders ${SDK_XPLANE}/CHeaders/XPLM)
endif()

if(WIN32)
  if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_COMPILER_IS_GNUCC)
    set(CMAKE_RC_COMPILER_INIT i686-w64-mingw32-windres)
    SET(CMAKE_RC_COMPILE_OBJECT "<CMAKE_RC_COMPILER> -O coff <DEFINES> -i <SOURCE> -o <OBJECT>")
  endif()
  ENABLE_LANGUAGE(RC)
endif(WIN32)

if(SDK_FSUIPC AND WIN32)
    include_directories(${SDK_FSUIPC})
    link_directories(${SDK_FSUIPC})
endif()

opentrack_module(opentrack-compat compat)
opentrack_module(opentrack-xplane-plugin x-plane-plugin)
#opentrack_module(freetrackclient freetrackclient)

if(SDK_XPLANE)
    # probably librt already included
    add_library(opentrack-xplane-plugin SHARED ${opentrack-xplane-plugin-c})
    if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_COMPILER_IS_GNUCC AND NOT APPLE)
        SET_TARGET_PROPERTIES(opentrack-xplane-plugin
            PROPERTIES LINK_FLAGS
            "-Wl,--version-script=${CMAKE_SOURCE_DIR}/x-plane-plugin/version-script.txt -shared -rdynamic -nodefaultlibs -undefined_warning -fPIC"
            COMPILE_FLAGS "-Wall -O2 -pipe -fPIC -DLIN -DXPLM210"
            LIBRARY_OUTPUT_NAME "opentrack.xpl"
            PREFIX "" SUFFIX "")
    endif()
    if(APPLE)
        SET_TARGET_PROPERTIES(opentrack-xplane-plugin PROPERTIES
                              COMPILE_FLAGS "-iframework ${SDK_XPLANE}/Libraries/Mac/ -DAPL -DXPLM210 -framework XPLM -framework XPWidgets"
                              LINK_FLAGS "-F${SDK_XPLANE}/Libraries/Mac/ -framework XPLM -framework XPWidgets")
    endif()
    if(UNIX AND NOT APPLE)
        target_link_libraries(opentrack-xplane-plugin rt)
    endif()
endif()

add_library(opentrack-compat STATIC ${opentrack-compat-c})
if(NOT WIN32 AND NOT APPLE)
    target_link_libraries(opentrack-compat rt)
endif()

opentrack_module(opentrack-csv csv)
add_library(opentrack-csv STATIC ${opentrack-csv-c})
target_link_libraries(opentrack-csv ${MY_QT_LIBS})

opentrack_module(opentrack-pose-widget pose-widget)
include_directories(pose-widget/) # else Qt moc breaks
opentrack_qt(opentrack-pose-widget)

add_library(opentrack-pose-widget STATIC ${opentrack-pose-widget-all})
target_link_libraries(opentrack-pose-widget ${MY_QT_LIBS})

opentrack_module(opentrack-spline-widget qfunctionconfigurator)
include_directories(qfunctionconfigurator/)
opentrack_qt(opentrack-spline-widget)
add_library(opentrack-spline-widget STATIC ${opentrack-spline-widget-all})
target_link_libraries(opentrack-spline-widget ${MY_QT_LIBS})

add_library(opentrack-version STATIC opentrack/version.C)
set_target_properties(opentrack-version PROPERTIES
    COMPILE_DEFINITIONS
    "IN_VERSION_UNIT;OPENTRACK_VERSION=\"${OPENTRACK__COMMIT}\"")

opentrack_library(opentrack-filter-accela ftnoir_filter_accela)
opentrack_library(opentrack-filter-kalman ftnoir_filter_kalman)
opentrack_library(opentrack-filter-ewma ftnoir_filter_ewma2)

opentrack_library(opentrack-proto-fgfs ftnoir_protocol_fg)

if(SDK_VJOY)
    include_directories(${SDK_VJOY})
    set(link-flags)
    if(CMAKE_C_COMPILER_IS_GNUCC)
        set(link-flags "-Wl,--enable-stdcall-fixup")
    endif()
    opentrack_library(opentrack-proto-vjoy ftnoir_protocol_vjoy LINK ${link-flags})
    target_link_libraries(opentrack-proto-vjoy ${MY_QT_LIBS} ${SDK_VJOY}/VJoy.dll)
endif()

if(SDK_ENABLE_LIBEVDEV)
    opentrack_library(opentrack-proto-libevdev ftnoir_protocol_libevdev)
    pkg_check_modules(libevdev REQUIRED QUIET libevdev)
    include_directories(${libevdev_INCLUDE_DIRS})
    target_link_libraries(opentrack-proto-libevdev ${libevdev_LIBRARIES})
endif()

if(SDK_FSUIPC)
    opentrack_library(opentrack-proto-fsuipc ftnoir_protocol_fsuipc)
    target_link_libraries(opentrack-proto-fsuipc ${SDK_FSUIPC}/FSUIPC_User.lib)
endif()

if(SDK_SIMCONNECT)
    opentrack_library(opentrack-proto-simconnect ftnoir_protocol_sc)
    include_directories(${SDK_SIMCONNECT}/inc)
    target_link_libraries(opentrack-proto-simconnect ${SDK_SIMCONNECT}/lib/SimConnect.lib)
endif()

if(WIN32)
    opentrack_library(opentrack-proto-freetrack ftnoir_protocol_ft)
    target_link_libraries(opentrack-proto-freetrack opentrack-csv opentrack-compat)
    opentrack_library(opentrack-proto-win32-mouse ftnoir_protocol_mouse)
endif()

opentrack_library(opentrack-proto-udp ftnoir_protocol_ftn)

if(WIN32)
    opentrack_library(opentrack-tracker-joystick ftnoir_tracker_joystick)
endif()

if(SDK_WINE_PREFIX)
    opentrack_library(opentrack-proto-wine ftnoir_protocol_wine)
    target_link_libraries(opentrack-proto-wine opentrack-compat opentrack-csv)
    if(NOT SDK_WINE_NO_WRAPPER)
        set(my-rt -lrt)
        if(APPLE)
            set(my-rt)
        endif()
        file(GLOB wine-deps ${CMAKE_SOURCE_DIR}/ftnoir_protocol_wine/*.cxx)
        add_custom_command(
          OUTPUT opentrack-wrapper-wine.exe.so
          DEPENDS ${wine-deps}
          COMMAND ${SDK_WINE_PREFIX}/bin/wineg++ -g -O2 -m32 -std=c++11 -o
                  opentrack-wrapper-wine.exe -I "${CMAKE_SOURCE_DIR}"
                  ${CMAKE_SOURCE_DIR}/ftnoir_protocol_wine/opentrack-wrapper-wine-main.cxx
                  ${CMAKE_SOURCE_DIR}/ftnoir_protocol_wine/opentrack-wrapper-wine-posix.cxx
                  ${CMAKE_SOURCE_DIR}/ftnoir_protocol_wine/opentrack-wrapper-wine-windows.cxx
                  ${my-rt})
        add_custom_target(wine-wrapper ALL DEPENDS opentrack-wrapper-wine.exe.so)
        add_dependencies(opentrack-proto-wine wine-wrapper)
        add_dependencies(wine-wrapper opentrack-compat)
    endif()
endif()

opentrack_library(opentrack-tracker-ht ftnoir_tracker_ht)
target_link_libraries(opentrack-tracker-ht opentrack-compat)

opentrack_library(opentrack-tracker-udp ftnoir_tracker_udp)
opentrack_library(opentrack-tracker-freepie-udp ftnoir_tracker_freepie-udp)

if(OpenCV_FOUND)
    opentrack_library(opentrack-tracker-pt ftnoir_tracker_pt)
    target_link_libraries(opentrack-tracker-pt ${OpenCV_LIBS})
    if(SDK_ARUCO_LIBPATH)
        opentrack_library(opentrack-tracker-aruco ftnoir_tracker_aruco)
        target_link_libraries(opentrack-tracker-aruco ${SDK_ARUCO_LIBPATH} ${OpenCV_LIBS})
    endif()
    target_link_libraries(opentrack-filter-kalman ${OpenCV_LIBS})
endif()

link_with_dinput8(opentrack-tracker-ht)
link_with_dinput8(opentrack-tracker-joystick)
link_with_dinput8(opentrack-tracker-pt)

if(SDK_RIFT)
    include_directories(${SDK_RIFT}/Include)
    include_directories(${SDK_RIFT}/Src)
    set(link-flags)
    set(c-flags)
    if(APPLE)
        set(link-flags "-framework CoreFoundation -framework CoreGraphics -framework IOKit -framework Quartz")
        set(c-flags "-fno-strict-aliasing")
    else()
        set(c-flags "-fno-strict-aliasing")
    endif()
    opentrack_library(opentrack-tracker-rift ftnoir_tracker_rift LINK ${link-flags} COMPILE ${c-flags})
    if(WIN32)
        target_link_libraries(opentrack-tracker-rift ${SDK_RIFT}/libLibOVR.a winmm setupapi ws2_32 imagehlp wbemuuid)
    else()
        if(NOT APPLE)
                target_link_libraries(opentrack-tracker-rift ${SDK_RIFT}/libLibOVR.a udev Xinerama)
        else()
                target_link_libraries(opentrack-tracker-rift ${SDK_RIFT}/libLibOVR.a)
        endif()
    endif()
endif()

if(SDK_HYDRA)
    include_directories(${SDK_HYDRA}/include)
    include_directories(${SDK_HYDRA}/include/sixense_utils)
    opentrack_library(opentrack-tracker-hydra ftnoir_tracker_hydra)
    if(WIN32)
        target_link_libraries(opentrack-tracker-hydra
            "${SDK_HYDRA}/lib/win32/release_dll/sixense.lib"
            "${SDK_HYDRA}/lib/win32/release_dll/sixense_utils.lib")
                    install(FILES "${SDK_HYDRA}/bin/win32/release_dll/sixense.dll" "${SDK_HYDRA}/bin/win32/release_dll/sixense_utils.dll" DESTINATION .)
            else()
                    if(SDK_HYDRA_AMD64)
                            set(underscore-sixtyfour _x64)
                    else()
                            set(underscore-sixtyfour)
                    endif()
                    if(APPLE)
                            set(underscore-dll _dll)
                            set(soext dylib)
                            set(sixense-plat osx)
                    else()
                            set(underscore-dll)
                            set(soext so)
                            set(sixense-plat linux)
                    endif()
                    install(FILES
                            "${SDK_HYDRA}/lib/${sixense-plat}${underscore-sixtyfour}/release${underscore-dll}/libsixense${underscore-sixtyfour}.${soext}"
                            "${SDK_HYDRA}/lib/${sixense-plat}${underscore-sixtyfour}/release${underscore-dll}/libsixense_utils${underscore-sixtyfour}.${soext}"
                            DESTINATION .
                    )
                    target_link_libraries(opentrack-tracker-hydra "${SDK_HYDRA}/lib/${sixense-plat}${underscore-sixtyfour}/release${underscore-dll}/libsixense${underscore-sixtyfour}.${soext}" "${SDK_HYDRA}/lib/${sixense-plat}${underscore-sixtyfour}/release${underscore-dll}/libsixense_utils${underscore-sixtyfour}.${soext}")
    endif()
endif()

if(UNIX OR APPLE)
    # Qxt bundled :: sorry for this, but gentoo ebuild is broken
    include_directories("qxt-mini/")
    if(APPLE)
        set(qxt-plat mac)
    else()
        set(qxt-plat x11)
    endif()
    file(GLOB qxt-mini-c qxt-mini/*.h qxt-mini/qxtglobalshortcut.cpp qxt-mini/plat/qxtglobalshortcut_${qxt-plat}.cpp)
    opentrack_qt(qxt-mini)
    add_library(opentrack-qxt-mini STATIC ${qxt-mini-all})
    target_link_libraries(opentrack-qxt-mini ${MY_QT_LIBS})
    if(NOT APPLE)
            target_link_libraries(opentrack-qxt-mini X11)
    endif()
endif()

if(WIN32 AND NOT SDK_CONSOLE_DEBUG)
    set(opentrack-win32-executable WIN32)
else()
    set(opentrack-win32-executable "")
endif()

opentrack_module(opentrack facetracknoir)
if(UNIX OR APPLE)
    list(APPEND opentrack-c qxt-mini/qxtglobalshortcut.h)
endif()
opentrack_qt(opentrack)
add_executable(opentrack ${opentrack-win32-executable} ${opentrack-all})

if(NOT WIN32)
    set_target_properties(opentrack PROPERTIES SUFFIX ".bin")
endif()
target_link_libraries(opentrack opentrack-api opentrack-version opentrack-pose-widget opentrack-spline-widget)



if(APPLE)
    SET_TARGET_PROPERTIES(opentrack-qxt-mini PROPERTIES LINK_FLAGS "-framework Carbon -framework CoreFoundation")
endif()

if(UNIX OR APPLE)
    target_link_libraries(opentrack opentrack-qxt-mini)
endif()

link_with_dinput8(opentrack)
target_link_libraries(opentrack ${MY_QT_LIBS})

if(CMAKE_SYSTEM STREQUAL LINUX)
    link_libraries(rt)
endif()

# ----

# make install

# ----

install(FILES ${CMAKE_SOURCE_DIR}/README.md DESTINATION .)

if(SDK_XPLANE)
    install(TARGETS opentrack-xplane-plugin RUNTIME DESTINATION . LIBRARY DESTINATION .  )
endif()

if(WIN32)
    install(DIRECTORY ${CMAKE_SOURCE_DIR}/bin/tracker-ht DESTINATION .)
    #install(TARGETS freetrackclient RUNTIME DESTINATION . LIBRARY DESTINATION .  )
endif()

install(DIRECTORY ${CMAKE_SOURCE_DIR}/3rdparty-notices DESTINATION .)

install(FILES "${CMAKE_SOURCE_DIR}/bin/freetrackclient.dll" DESTINATION .)
install(FILES "${CMAKE_SOURCE_DIR}/bin/NPClient.dll" "${CMAKE_SOURCE_DIR}/bin/NPClient64.dll" "${CMAKE_SOURCE_DIR}/bin/TrackIR.exe" DESTINATION .)
install(DIRECTORY "${CMAKE_SOURCE_DIR}/bin/settings" "${CMAKE_SOURCE_DIR}/clientfiles" DESTINATION .)

if(NOT WIN32 AND SDK_WINE_PREFIX AND NOT SDK_WINE_NO_WRAPPER)
    install(FILES "${CMAKE_BINARY_DIR}/opentrack-wrapper-wine.exe.so"
            DESTINATION .)
endif()

install(TARGETS opentrack DESTINATION .)

if(SDK_VJOY)
    install(FILES "${SDK_VJOY}/VJoy.dll" DESTINATION .)
endif()

if(WIN32)
    install(FILES "${CMAKE_SOURCE_DIR}/bin/cleye.config" DESTINATION .)
endif()
