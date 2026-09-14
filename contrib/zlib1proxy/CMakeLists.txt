# Create a proxy DLL named "zlib1.dll" that simply forwards calls to the main
# ZLIB::ZLIB DLL.
#
# Nothing is more permanent than a temporary solution that works.
if(NOT ZLIB_BUILD_SHARED)
    message(
        FATAL_ERROR
            "ZLIB_BUILD_ZLIB1_PROXY_DLL requires ZLIB_BUILD_SHARED to be ON")
endif()
if(NOT (WIN32 AND MSVC))
    message(
        FATAL_ERROR
            "ZLIB_BUILD_ZLIB1_PROXY_DLL needs an MSVC-compatible linker")
endif()
if(CMAKE_VERSION VERSION_LESS 3.15)
    message(
        FATAL_ERROR
            # We use the TARGET_FILE_BASE_NAME generator expression.
            "ZLIB_BUILD_ZLIB1_PROXY_DLL requires CMake >= 3.15" "(for the )")
endif()

# Extract the exported symbol names from the canonical zlib.def.
file(STRINGS ${zlib_SOURCE_DIR}/win32/zlib.def ZLIB1PROXY_DEF_LINES)
set(ZLIB1PROXY_SYMBOLS)
foreach(line IN LISTS ZLIB1PROXY_DEF_LINES)
    string(STRIP "${line}" line)
    if(line MATCHES "^;"
       OR line STREQUAL ""
       OR line STREQUAL "EXPORTS")
        continue()
    endif()
    list(APPEND ZLIB1PROXY_SYMBOLS "${line}")
endforeach()

set(ZLIB1PROXY_DEF_CONTENT
    "; Generated -- forwards every zlib export to the real ZLIB::ZLIB DLL\nLIBRARY zlib1\nEXPORTS\n"
)
foreach(sym IN LISTS ZLIB1PROXY_SYMBOLS)
    string(APPEND ZLIB1PROXY_DEF_CONTENT
           "    ${sym}=$<TARGET_FILE_BASE_NAME:zlib>.${sym}\n")
endforeach()

set(ZLIB1PROXY_DEF_FILE ${zlib_BINARY_DIR}/zlib1proxy_$<CONFIG>.def)
file(
    GENERATE
    OUTPUT ${ZLIB1PROXY_DEF_FILE}
    CONTENT "${ZLIB1PROXY_DEF_CONTENT}")

set(ZLIB1PROXY_STUB_SRC ${zlib_BINARY_DIR}/zlib1proxy_stub.c)
file(
    WRITE ${ZLIB1PROXY_STUB_SRC}
    "#include <windows.h>\n"
    "BOOL WINAPI DllMain(HINSTANCE inst, DWORD reason, LPVOID reserved) {\n"
    "    (void)inst; (void)reason; (void)reserved;\n"
    "    return TRUE;\n"
    "}\n")

add_library(zlib1proxy SHARED ${ZLIB1PROXY_STUB_SRC})
add_library(ZLIB::ZLIB1PROXY ALIAS zlib1proxy)

# The linker only turns a "name=module.name" EXPORTS entry into a real forwarder
# RVA if it can resolve "module.name" itself -- which means the real DLL's
# import library has to be a link input here. Without this, LINK falls back to
# looking for a locally-defined "name" and fails with LNK2001 for every
# forwarded symbol.
target_link_libraries(zlib1proxy PRIVATE ZLIB::ZLIB)

set_target_properties(
    zlib1proxy
    PROPERTIES OUTPUT_NAME zlib1
               EXPORT_NAME ZLIB1PROXY
               LINK_DEPENDS ${ZLIB1PROXY_DEF_FILE})

target_link_options(zlib1proxy PRIVATE "/DEF:${ZLIB1PROXY_DEF_FILE}")

if(ZLIB_INSTALL)
    install(
        TARGETS zlib1proxy
        COMPONENT Runtime
        RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
        ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
        LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}")
endif(ZLIB_INSTALL)
