cmake_minimum_required(VERSION 3.20)
project(modern_lgp LANGUAGES CXX)
enable_testing()
# Native source has one canonical location, also included in Python wheels.
add_subdirectory(src/lgp/runtime/_native/cpp)
