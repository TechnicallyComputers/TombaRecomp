cmake_minimum_required(VERSION 3.20)
project(TombaRecomp LANGUAGES C CXX)

set(PSXRECOMP_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/psxrecomp" CACHE PATH
    "Path to psxrecomp")

# --- Opt-in product surfaces (OFF by default in the framework) -------------
# Scaffold prompts / flags fill these. Without recomp-ui, PSX_RECOMP_UI is OFF.
# Setup-host CI requires WIZARD_CMAKE_BLOCK / ENABLE_SETUP_WIZARD (FORCE_SETUP_HOST
# alone does not open first-run / Generate & rebuild).
# recomp-ui submodule present (PSX_RECOMP_UI defaults ON).
# Full netplay UI — enable after adding recomp-ui + testing:
# if(EXISTS "${PSXRECOMP_ROOT}/lib/recomp-net/CMakeLists.txt")
#     set(PSX_NETPLAY ON CACHE BOOL "…" FORCE)
# endif()
set(PSX_SETUP_WIZARD ON CACHE BOOL
    "Advertise first-run setup wizard + Generate & rebuild in recomp-ui" FORCE)
include("${PSXRECOMP_ROOT}/runtime/runtime.cmake")

# Trusted native implementations behind the preloaded mod catalog below. The
# manifests select these by plugin id; archives never carry native code.
#
# EXTRAS_SOURCES rather than target_sources(psx-runtime ...): psxrecomp_add_
# runtime_target also builds the <exe>_pgxp A/B sibling from the SAME argument
# list, so a plugin added to the target after the fact would be missing from
# that build. cmake_parse_arguments accumulates a repeated multi-value keyword,
# so this merges with the CODEGEN_SETUP_SOURCES the framework passes under the
# same keyword rather than displacing them.
file(GLOB TOMBA_MOD_PLUGIN_SOURCES CONFIGURE_DEPENDS
    "${CMAKE_CURRENT_SOURCE_DIR}/src/mods/*.c")

psxrecomp_add_game_runtime(psx-runtime
    # ENABLE_NETPLAY_IF_PRESENT
    # NETPLAY_LOBBY_URL "ws://netplay.retcomm.net:8765"
    ENABLE_SETUP_WIZARD
    WINDOW_TITLE "Tomba! Recompiled"
    DEFAULT_GAME_CONFIG_PATH "game.toml"
    MAX_PLAYERS 1
    LAUNCHER_BOXART "${CMAKE_CURRENT_SOURCE_DIR}/launcher_assets/img/boxart.tga"
    APP_ICON "${CMAKE_CURRENT_SOURCE_DIR}/assets/psxrecomp.ico"
    GEN_MARKER "generated/SCUS_942.36_dispatch.c"
    GEN_FULL_GLOB "generated/SCUS_942.36_full_*.c"
    CODEGEN_SETUP_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/codegen_setup.c"
    EXTRAS_SOURCES ${TOMBA_MOD_PLUGIN_SOURCES}
)

# Stage runtime configs next to the exe for direct build/ launches. Only copy
# files that exist — never fail POST_BUILD when game_options.toml is absent.
foreach(_PSX_RUNTIME_CONFIG game.toml game_options.toml)
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/${_PSX_RUNTIME_CONFIG}")
        add_custom_command(TARGET psx-runtime POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_if_different
                "${CMAKE_CURRENT_SOURCE_DIR}/${_PSX_RUNTIME_CONFIG}"
                "$<TARGET_FILE_DIR:psx-runtime>/${_PSX_RUNTIME_CONFIG}"
            COMMENT "Staging ${_PSX_RUNTIME_CONFIG}"
            VERBATIM)
    endif()
endforeach()

# Preload the game-owned mod catalog. Features stay disabled until selected in
# the launcher, and plugin ids resolve only to the implementations compiled
# above. This runs AFTER the framework's own POST_BUILD staging, which clears
# mods/packages and lays down its builtin catalog first — copy_directory
# merges, so both catalogs end up side by side.
#
# Scoped to packages/ on purpose: mods/state.toml next to it is USER STATE
# (which mods are enabled, with which options) written at run time, and
# mods/preloaded/README.md is developer documentation that does not belong
# beside the exe.
set(TOMBA_PRELOADED_MODS "${CMAKE_CURRENT_SOURCE_DIR}/mods/preloaded/packages")
if(EXISTS "${TOMBA_PRELOADED_MODS}")
    add_custom_command(TARGET psx-runtime POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_directory
            "${TOMBA_PRELOADED_MODS}"
            "$<TARGET_FILE_DIR:psx-runtime>/mods/packages"
        COMMENT "Preloading the Tomba mod catalog"
        VERBATIM)
endif()
