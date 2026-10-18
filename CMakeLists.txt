cmake_minimum_required(VERSION 3.20)

# Turok: Evolution, statically recompiled on the xboxrecomp toolkit.
#
# Build with scripts/build.py, which lifts the game first. The generated
# sources in src/recomp/gen/ come from your own copy of the game and are
# never committed.

project(turok_recomp C CXX)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

add_compile_definitions(
    WIN32_LEAN_AND_MEAN
    NOMINMAX
)

set(XBOXRECOMP_DIR "${CMAKE_CURRENT_SOURCE_DIR}/xboxrecomp" CACHE PATH
    "Path to the xboxrecomp toolkit (the submodule by default)")

if(NOT EXISTS "${XBOXRECOMP_DIR}/CMakeLists.txt")
    message(FATAL_ERROR "xboxrecomp not found at ${XBOXRECOMP_DIR}. "
        "Run: git submodule update --init --recursive")
endif()

# Checks that every call preserves ebx/esi/edi and esp. Set before
# add_subdirectory: the check is emitted into the generated game code but
# its logger lives in xbox_kernel. See the toolkit's templates/new-game.
option(RECOMP_ABI_CHECK "Verify callee-saved registers across every call" OFF)
if(RECOMP_ABI_CHECK)
    add_compile_definitions(RECOMP_ABI_CHECK)
endif()

add_subdirectory(${XBOXRECOMP_DIR} ${CMAKE_BINARY_DIR}/xboxrecomp)

# CONFIGURE_DEPENDS so a re-lift that adds or removes a chunk is picked up
# without a manual re-configure.
file(GLOB RECOMP_GEN_SOURCES CONFIGURE_DEPENDS "src/recomp/gen/*.c")
if(NOT RECOMP_GEN_SOURCES)
    message(FATAL_ERROR "No generated sources in src/recomp/gen. "
        "Lift the game first: py -3 scripts/build.py")
endif()

add_executable(${PROJECT_NAME} WIN32
    src/main.c
    src/recomp_manual.c
    ${RECOMP_GEN_SOURCES}
)

target_link_libraries(${PROJECT_NAME} PRIVATE xboxrecomp)

target_include_directories(${PROJECT_NAME} PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/src
    ${CMAKE_CURRENT_SOURCE_DIR}/src/recomp
    ${XBOXRECOMP_DIR}/include
)

target_link_libraries(${PROJECT_NAME} PRIVATE
    d3d11
    dxgi
    dxguid
    xinput
    winmm
    dbghelp     # lets the crash handler name the generated function that faulted
)

if(MSVC)
    # /DEBUG: a PDB in every configuration, so a crash names sub_XXXXXXXX.
    target_link_options(${PROJECT_NAME} PRIVATE /LARGEADDRESSAWARE /DEBUG)
    target_compile_options(${PROJECT_NAME} PRIVATE /bigobj /MP)
    # Lifted code reads ebp before writing it by design (fpo_leaf functions
    # inherit the caller's frame), which /RTC1 aborts on through a message box.
    string(REGEX REPLACE "/RTC[1csu]+" "" CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG}")
endif()

# Turok has no 16:9 mode of its own (no widescreen flag at CreateDevice, no
# read of the console's video flags), so widescreen means Hor+: its vertex
# programs take oPos.x from c96, the first row of the model-view-projection
# the game builds for each object, and the toolkit scales it on screen draws.
# The front end and the movies are shown 4:3 (src/recomp_manual.c).
if(COMMAND recomp_title_widescreen)
    recomp_title_widescreen(HOR_PLUS REGISTER 96)
endif()

# The settings window: video options, input bindings, then starts the game.
if(COMMAND recomp_add_launcher)
    recomp_add_launcher(${PROJECT_NAME}_launcher ${PROJECT_NAME}.exe)
endif()
