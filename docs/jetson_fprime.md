# F Prime on the Jetson AGX Orin

SCALES has developed an F Prime deployment for the Jetson AGX Orin that can be found in our [fprime-scales-ref](https://github.com/BroncoSpace-Lab/fprime-scales-ref/tree/main) GitHub in JetsonDeployment.

## How to Clone

There are a few git submodules used here, so when cloning be sure to init and update them.

```
git clone https://github.com/BroncoSpace-Lab/fprime-scales-ref.git
cd fprime-scales-ref
make setup
make arena-init
source fprime-venv/bin/activate
```

### Necessary Changes

Some lines need to be commented in `lib/fprime/cmake/API.cmake` in order to use `fprime-python`. Comment out lines [545](https://github.com/nasa/fprime/blob/5a3b873854fe4d646d6874d134585535652fddb9/cmake/API.cmake#L545) and [562](https://github.com/nasa/fprime/blob/5a3b873854fe4d646d6874d134585535652fddb9/cmake/API.cmake#L562).

After this, you should be good to go!

---

## JetsonDeployment Build Configuration

You must generate build JetsonDeployment on the Jetson, we have not set up cross-compilation for aarch64-linux yet.

<details>

<summary> JetsonDeployment Build Configuration Details </summary>

Your `settings.ini` should look like this:

```
[fprime]
project_root: .
framework_path:     ./lib/fprime
; uncomment this line for JetsonDeployment
library_locations:  ./lib/fprime-python:./lib/fprime-scales
; uncomment this line for ImxDeployment
; library_locations:  ./lib/fprime-scales:

default_cmake_options:  FPRIME_ENABLE_FRAMEWORK_UTS=OFF
                        FPRIME_ENABLE_AUTOCODER_UTS=OFF
```

Your `project.cmake` should look like this:

```
# This CMake file is intended to register project-wide objects.
# This allows for reuse between deployments, or other projects.

add_fprime_subdirectory("${CMAKE_CURRENT_LIST_DIR}/Components")
# add_fprime_subdirectory("${CMAKE_CURRENT_LIST_DIR}/ImxDeployment/")
add_fprime_subdirectory("${CMAKE_CURRENT_LIST_DIR}/JetsonDeployment/")
add_fprime_subdirectory("${CMAKE_CURRENT_LIST_DIR}/lib/")
# add_fprime_subdirectory("${CMAKE_CURRENT_LIST_DIR}/lib/fprime-scales/scales/scalesSvc")
```

Your `CMakeLists.txt` in the root project directory should have line 17 containing `register_fprime_target("${CMAKE_SOURCE_DIR}/lib/fprime-python/cmake/target/pybind.cmake")` **uncommented**.

Your `Components/CMakeLists.txt` should look like this:

```
# Include project-wide components here

# add_fprime_subdirectory("${CMAKE_CURRENT_LIST_DIR}/StandardBlankComponent/")
# add_fprime_subdirectory("${CMAKE_CURRENT_LIST_DIR}/PythonComponent/")
add_fprime_subdirectory("${CMAKE_CURRENT_LIST_DIR}/MLComponent/")
add_fprime_subdirectory("${CMAKE_CURRENT_LIST_DIR}/RunLucidCamera/")
```

</details>

After all of this, **on the Jetson** you should be able to generate and build the JetsonDeployment.

To generate: 

```
fprime-util generate aarch64-linux -f
```

To build and set up the build environment:

```
make build-jetson
```