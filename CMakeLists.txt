cmake_minimum_required(VERSION 3.16)

project(picowal_retailprimitives C)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

set(PICOWAL_ROOT "${CMAKE_CURRENT_LIST_DIR}/../picowal" CACHE PATH "Path to picowal checkout")
set(PICOWEB_ROOT "${CMAKE_CURRENT_LIST_DIR}/../picoweb" CACHE PATH "Path to picoweb checkout")

add_library(picowal_retailprimitives
    src/picowal_retail.c
    src/picowal_retail_web.c
)

target_include_directories(picowal_retailprimitives PUBLIC
    ${CMAKE_CURRENT_LIST_DIR}/src
    ${PICOWAL_ROOT}/src
    ${PICOWEB_ROOT}/include
)

target_compile_definitions(picowal_retailprimitives PUBLIC
    PICOWAL_HOST=1
    PICOWAL_NO_DEFAULT_STORE=1
)

add_executable(picowal_retail_smoke
    ${PICOWAL_ROOT}/src/picowal_api.c
    ${PICOWAL_ROOT}/src/picowal_search.c
    ${PICOWAL_ROOT}/src/picowal_store_fs.c
    host/tests/retail_smoke.c
)

target_link_libraries(picowal_retail_smoke PRIVATE picowal_retailprimitives)

if(UNIX)
    target_link_libraries(picowal_retail_smoke PRIVATE m)
endif()
