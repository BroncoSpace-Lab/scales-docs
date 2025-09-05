# SCALES Demo Development

This demo combines the F Prime Hub Pattern developed by the SCALES team with COTS evaluation boards for the Flight Computer and Edge Computer. The demo aims to accomplish the following:

- Test the capability of the split computing architecture
- Test the capability of using a network ethernet switch in this architecture
- Test the command and data handling aspects of the Flight Computer
- Test using the Hub Pattern to have the Flight Computer command the Edge Computer

Watch our video demo on [YouTube](https://youtu.be/-g3Wv_fr9r8?si=2xow8_22aNjE1XDO)!

Code and reference deployment can be found on our [fprime-scales-ref](https://github.com/BroncoSpace-Lab/fprime-scales-ref/tree/main) GitHub.

## Setup

The i.MX8X Flight Computer evaluation board, Jetson Edge Computer evaluation board, and COTS ethernet camera will be connected through a COTS managed ethernet switch. The i.MX will command the Jetson through the Hub Pattern to take a picture using the Ethernet Camera. The picture will be saved on the Jetson. When completed, the i.MX will command the Jetson through the Hub Pattern to run a computer vision algorithm on the images taken with the camera. The results will be sent to the i.MX.

## Process

Using a pre-existing F Prime deployment, we created a new component called RunLucidCamera. The component achieves the following:

- GDS command to set up the camera and make sure it is connected
- GDS command to take a picture and save an image from the ethernet camera

We recycled and modified existing example code from the ethernet camera's SDK. To do this, we needed to integrate the Arena SDK for the ethernet camera into the F Prime deployment. 

### Arena SDK in F Prime

The Arena SDK was installed from [Lucid](https://thinklucid.com/downloads-hub/)

1. Create a `lib` folder in the root directory of the project.
2. In `fprime-scales-ref/project.cmake` add the `lib` folder as a subdirectory.
3. Copy the Arena SDK folders into `lib` at `fprime-scales-ref/lib/ArenaSDK`
4. Make a CMakeLists.txt files at `fprime-scales-ref/lib/CMakeLists/txt` and add the following: `add_fprime_subdirectory("${CMAKE_CURRENT_LIST_DIR}/ArenaSDK")`
5. Make a new CMakeLists.txt at `fprime-scales-ref/lib/ArenaSDK/CMakeLists/txt` and add the Arena SDK library paths. Currently, the only way this worked was to add the paths to the exact files we wanted. 

    ```
    set(MODULE_NAME lib_ArenaSDK)

    add_library(${MODULE_NAME} INTERFACE)
    target_include_directories(${MODULE_NAME} INTERFACE
        ${CMAKE_CURRENT_LIST_DIR}/include/Arena
        ${CMAKE_CURRENT_LIST_DIR}/GenICam/library/CPP/include
        ${CMAKE_CURRENT_LIST_DIR}/include/Save
        ${CMAKE_CURRENT_LIST_DIR}/include/GenTL
        )

    target_link_libraries(${MODULE_NAME} INTERFACE
        # Your existing libraries
        Components_RunLucidCamera
        
        # Arena SDK libraries - provide full paths to .so or .a files
        ${CMAKE_CURRENT_LIST_DIR}/lib/libarena.so          # Core Arena library
        ${CMAKE_CURRENT_LIST_DIR}/lib/libarena.so.0          # Core Arena library
        ${CMAKE_CURRENT_LIST_DIR}/lib/libsavec.so          # Save functionality
        ${CMAKE_CURRENT_LIST_DIR}/lib/libgentl.so          # GenTL functionality
        ${CMAKE_CURRENT_LIST_DIR}/lib/liblucidlog.so       # Lucid logging
        ${CMAKE_CURRENT_LIST_DIR}/lib/libsave.so
        ${CMAKE_CURRENT_LIST_DIR}/lib/libarenac.so

        ${CMAKE_CURRENT_LIST_DIR}/ffmpeg/libavcodec.so
        ${CMAKE_CURRENT_LIST_DIR}/ffmpeg/libavformat.so
        ${CMAKE_CURRENT_LIST_DIR}/ffmpeg/libavutil.so
        ${CMAKE_CURRENT_LIST_DIR}/ffmpeg/libswresample.so


        # GenICam libraries
        ${CMAKE_CURRENT_LIST_DIR}/GenICam/library/lib/Linux64_ARM/libGenApi_gcc54_v3_3_LUCID.so
        ${CMAKE_CURRENT_LIST_DIR}/GenICam/library/lib/Linux64_ARM/libGCBase_gcc54_v3_3_LUCID.so
        ${CMAKE_CURRENT_LIST_DIR}/GenICam/library/lib/Linux64_ARM/libMathParser_gcc54_v3_3_LUCID.so
        ${CMAKE_CURRENT_LIST_DIR}/GenICam/library/lib/Linux64_ARM/liblog4cpp_gcc54_v3_3_LUCID.so
        ${CMAKE_CURRENT_LIST_DIR}/GenICam/library/lib/Linux64_ARM/libLog_gcc54_v3_3_LUCID.so
        ${CMAKE_CURRENT_LIST_DIR}/GenICam/library/lib/Linux64_ARM/libNodeMapData_gcc54_v3_3_LUCID.so
        ${CMAKE_CURRENT_LIST_DIR}/GenICam/library/lib/Linux64_ARM/libXmlParser_gcc54_v3_3_LUCID.so
    )
    ```

6. In `fprime-scales-ref/Components/RunLucidCamera/RunLucidCamera.cpp`, update the include path to `#include "ArenaApi.h"` and `#include SaveApi.h`, and integrate the `CppSave_Png.cpp` example code from the Arena SDK into the `RunLucidCamera.cpp`. The constructor and deconstructor for the component was modified to include parts of the Arena SDK example code. We also modified the way the example code was saving the images so the old images don't get overwritten. The current code is [here](https://github.com/BroncoSpace-Lab/fprime-scales-ref/tree/main/Components/RunLucidCamera). 

---

# To Run the SCALES Demo

SCALES has developed an F Prime project for this demo that can be found in our [fprime-scales-ref](https://github.com/BroncoSpace-Lab/fprime-scales-ref/tree/main) GitHub.

## IMX Setup

1. Follow the instructions above to build ImxDeployment on the host machine. Use the following command to ssh into the IMX.

    ```
    ssh root@<ip of imx> -o HostKeyAlgorithms=+ssh-rsa -o PubKeyAcceptedAlgorithms=+ssh-rsa
    ```

2. Make sure you are able to ping both the host machine and the Jetson from the IMX. Copy the ImxDeployment binary from the host machine to the IMX. (Run this command on the host machine.)

    ```
    scp -oHostKeyAlgorithms=+ssh-rsa -oPubkeyAcceptedKeyTypes=+ssh-rsa ~/fprime-scales-ref/build-artifacts/imx8x/ImxDeployment/bin/ImxDeployment root@<ip of imx>:~/.
    ```

3. Copy the binary files for the sequences to the IMX.

    ```
    scp -oHostKeyAlgorithms=+ssh-rsa -oPubkeyAcceptedKeyTypes=+ssh-rsa ~/fprime-scales-ref/save-png.bin root@<ip of imx>:~/.
    scp -oHostKeyAlgorithms=+ssh-rsa -oPubkeyAcceptedKeyTypes=+ssh-rsa ~/fprime-scales-ref/batch-send-img.bin root@<ip of imx>:~/.
    ```

## Jetson Setup

1. On the Jetson, follow the above directions to generate and build JetsonDeployment.

2. Change the IP of the IMX in `Jetsondeployment/Top/JetsonDeploymentTopology.cpp` to match the IP of the IMX.

    ```
    // line 32
    const char* REMOTE_HUIP_ADDRESS = "10.3.2.2"; // ip of JPL IMX
    // const char* REMOTE_HUIP_ADDRESS = "192.168.0.66"; // ip of CPP IMX
    const U32 REMOTE_HUPORT = 50500;
    ```

3. Rebuild JetsonDeployment.

    ```
    make build-jetson
    ```

4. **For first time setup only:** Make a folder with a symbolic link to where the camera images are saved. This is done to assure the paths for commands in the fprime-gds are not too long.

    ```
    sudo ln -s ~/fprime-scales-ref/build-python-fprime-aarch64-linux/Images/ ./Images
    ```

    The `Images` folder will be created in your root directory.

## Host Setup

1. Open another terminal on the host machine and enter the directory for the repo and source your environment.

    ```
    cd fprime-scales-ref
    source fprime-venv/bin/activate
    ```

2. Copy the ImxDeployment dictionary to the GDS-Dictionary folder on the host machine. Run this command on the host machine.

    ```
    cp ~/fprime-scales-ref/build-artifacts/imx8x/ImxDeployment/dict/ImxDeploymentTopologyAppDictionary.xml ~/fprime-scales-ref/GDS-Dictionary/.
    ```

3. Copy the JetsonDeployment dictionary from the Jetson to the host machine. Run this command on the host machine.

    ```
    scp <jetson name>@<jetson IP>:~/fprime-scales-ref/build-artifacts/aarch64-linux/JetsonDeployment/dict/JetsonDeploymentTopologyAppDictionary.xml ~/fprime-scales-ref/GDS-Dictionary/.
    ```

6. Combine the GDS dictionaries with the `merger.py` script. Run this command on the host machine.

    ```
    python merger.py JetsonDeploymentTopologyAppDictionary.xml ImxDeploymentTopologyAppDictionary.xml GDSDictionary.xml
    ```

You are now ready to run the demo!

## Running the Demo

1. After you finished setting up the demo in the previous section, on the host machine, navigate to the `GDS-Dictionary` folder and run the fprime-gds.

    ```
    fprime-gds -n --dictionary GDSDictionary.xml --ip-client --ip-address <ip of imx>
    ```

2. On the IMX, run the ImxDeployment binary. You should see a green dot on the fprime-gds and "Accepted client" in the IMX terminal.

    ```
    ./ImxDeployment -a 0.0.0.0 -p 50000
    ```

3. On the Jetson, navigate to the `build-python-fprime-aarch64-linux` directory to run the fprime-gds using python.

    ```
    cd build-python-fprime-aarch64-linux
    python
    ```

    Once the python environment opens, run the following commands to connect to the IMX's fprime-gds using the hub pattern. If you want to exit the python environment, the command is `exit()`.

    ```
    import python_extension
    python_extension.main()
    ```

4. On the host machine, use the fprime-gds to run the `jetson_cmdDisp.CMD_NO_OP` to test the connection with the Jetson. Do the same for the IMX with the `imx_cmdDisp.CMD_NO_OP`. You can see both events and their status in the "Events" tab of the GDS.

5. Once the camera is connected, run the `jetson_lucidCamera.SETUP_CAMERA` command to verify the connection via fprime. 

6. To take a picture with the camera, run the `imx_cmdSeq.CD_RUN` command in the fprime-gds with argument `demo.bin`. This will take a pictire with the camera, downlink it to the IMX, and then downlink it again to the Host Machine. You can download the image from the `Downlink` tab in the GDS. This sequence will also run a resnet ML model to identify what is in the image. The output will be displayed in the Events tab of the GDS. Images are deleted from the Jetson after the `demo.bin` sequence concludes. Repeat this step if you wish to take more images.

That's how to run the SCALES demo!

Watch our video demo on [YouTube](https://youtu.be/-g3Wv_fr9r8?si=2xow8_22aNjE1XDO)!

---

This project was auto-generated by the F' utility tool. 

F´ (F Prime) is a component-driven framework that enables rapid development and deployment of spaceflight and other embedded software applications.
**Please Visit the F´ Website:** https://fprime.jpl.nasa.gov.