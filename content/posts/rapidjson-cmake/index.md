---
title: "Add RapidJSON with CMake"
date: 2019-12-09
summary: "RapidJSON is a nice C++ library for parsing JSON files. If you use CMake, we can automatically download it as a dependency for your project."
---

[RapidJSON](https://github.com/Tencent/rapidjson) is a cool and fast C++ library from Tencent for parsing JSON files.

Last time I used it, I wanted to include it with [CMake](https://cmake.org/).
However, I struggled a little with it.
So this post will firstly serve as a reminder for me and secondly make you hopeuflly struggle less than me.

# Project Layout

First of all, we assume a typical CMake project layout:

```
My_Project
├── CMakeLists.txt
├── include
│   └── ...
├── src
│   ├── my_project.cc
│   ├── ...
│   └── local.cmake
├── vendor
│   └── rapidjson.cmake
└── ...
```

Only relevant files for this post are depicted.

Of course, your project setup might differ, but now we can talk about a common baseline.


# Download RapidJSON via CMake

CMake can manage external sources as [external projects](https://cmake.org/cmake/help/latest/module/ExternalProject.html)
We now want to use CMake's `ExternalProject_Add` to automatically download and add RapidJSON.
Now the cool thing is that RapidJSON is a "header-only" library, meaning that we can simply copy and include the headers from `rapidjson/include` and we can then use everything.


Thus, our `rapidjson.cmake` will only download the library from GitHub and then set `RAPIDJSON_INCLUDE_DIR` that contains the path to the header files.

With that, our `rapidjson.cmake` looks like this:

```CMake
# Download RapidJSON
ExternalProject_Add(
    rapidjson
    PREFIX "vendor/rapidjson"
    GIT_REPOSITORY "https://github.com/Tencent/rapidjson.git"
    GIT_TAG f54b0e47a08782a6131cc3d60f94d038fa6e0a51
    TIMEOUT 10
    CMAKE_ARGS
        -DRAPIDJSON_BUILD_TESTS=OFF
        -DRAPIDJSON_BUILD_DOC=OFF
        -DRAPIDJSON_BUILD_EXAMPLES=OFF
    CONFIGURE_COMMAND ""
    BUILD_COMMAND ""
    INSTALL_COMMAND ""
    UPDATE_COMMAND ""
)

# Prepare RapidJSON (RapidJSON is a header-only library)
ExternalProject_Get_Property(rapidjson source_dir)
set(RAPIDJSON_INCLUDE_DIR ${source_dir}/include)
```


# Include RapidJSON in our Project

In `CMakeLists.txt` we now need to declare two things:

 1. The `rapidjson.cmake` file exists and should be executed.
 2. Add `RAPIDJSON_INCLUDE_DIR` to the list of included directories.

The first step is done by including the following line somewhere in the `CMakeLists.txt` file (you probably already have other `include()` there as well):

```CMake
include("${CMAKE_SOURCE_DIR}/vendor/rapidjson.cmake")
```

The next step is to add the header files from RapidJSON to the included directories. This can be done with CMake's `include_directories()`:

```CMake
include_directories(${RAPIDJSON_INCLUDE_DIR})
```

You might already have other included directories there.
In the end, the `CMakeLists.txt` might look similar to this:

```CMake
# CMakeLists.txt

project(my_project)
cmake_minimum_required(VERSION 3.7)

include("${CMAKE_SOURCE_DIR}/vendor/rapidjson.cmake")

include_directories(
    ${CMAKE_SOURCE_DIR}/include
    ${GFLAGS_INCLUDE_DIR}
    ${BENCHMARK_INCLUDE_DIR}
    ${RAPIDJSON_INCLUDE_DIR}
)
```


# Add RapidJSON as a Dependency

Now one thing is left before we can actually use RapidJSON.
Right now, we only tell CMake _what_ to do and not _when_ to do it.

Of course, we want to add RapidJSON as a dependency to our own source code so that it will be downloaded before our code is compiled.

Currently, our `src/local.cmake` might look like this:

```CMake
# Add source files
file(GLOB_RECURSE SRC_CC "src/*.cc")

# Compile my_project as a library
add_library(my_project SHARED ${SRC_CC})
# Link gflags to my_project
target_link_libraries(my_project gflags Threads::Threads)
```

Now add the following line under `add_library()` (or `add_executable()` in your case):

```CMake
...
add_library(my_project SHARED ${SRC_CC})
add_dependencies(my_project rapidjson)
...
```

Now CMake will first download RapidJSON and include its headers and then compile our own project.

In `my_project.cc` we can now import RapidJSON!

```C++
#include "rapidjson/document.h"
```

Now you should be able to run `make` and everything should compile.
