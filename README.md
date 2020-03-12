# pytorch-onnx-ncnn
## 1. pytorch->onnx
Import torch.onnx

    dummy_input = Variable(torch.randn(1, 3, 320, 256)).cuda()
    input_names = ["input0"]
    output_names = ["output0"]
    torch.onnx.export(pose_model, dummy_input, "pose.onnx", export_params=True, verbose=True, input_names=input_names, output_names=output_names)

Check  
Use pip to install protobuf,onnx  

    import onnx
    onnx_model = onnx.load(output_onnx)
    onnx.checker.check_model(onnx_model)  # assuming throw on error

Load model and test  
pip install onnxruntime  

    import onnxruntime
    session = onnxruntime.InferenceSession("pose-sim.onnx")
    input_name = session.get_inputs()[0].name
    output_name = session.get_outputs()[0].name
    #make sure that types of input and out are both numpy
    onnx_hm = session.run([output_name], {input_name: inps})#here inps is the input data

## 2. onnx->ncnn: compile with Windows 10 Visual Studio 2017
__2.1 install protobuf__  
Download and unzip protobuf-all-3.11.4.zip from https://github.com/protocolbuffers/protobuf/releases  
Use x64 Native Tools Command Prompt for VS 2017  

    cd <protobuf-root-dir>
    mkdir build-vs2017
    cd build-vs2017
    cmake -G"NMake Makefiles" -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=%cd%/install -Dprotobuf_BUILD_TESTS=OFF -Dprotobuf_MSVC_STATIC_RUNTIME=OFF ../cmake
    nmake
    nmake install

__2.2 install opencv4.__  
Download opencv-4.2.0-vc14_vc15.exe from https://opencv.org/ and unzip  
Add Path `E:\VS\opencv\build\x64\vc14\bin` to Windows  
copy `bin/opencv_world420.dll` and `bin/opencv_world420d.dll` to `C:\WINDOWS\SysWOW64`  

__2.3 install ncnn__  
git clone https://github.com/Tencent/ncnn.git  
modify `ncnn/examples/CMakelist.txt`  
add `set(OpenCV_DIR "E:/VS/opencv/build")`  

    cd <ncnn-root-dir>
    mkdir -p build-vs2017
    cd build-vs2017
    # cmake option NCNN_VULKAN for enabling vulkan
    cmake -G"NMake Makefiles" -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=%cd%/install -DProtobuf_INCLUDE_DIR=<protobuf-root-dir>/build-vs2017/install/include -DProtobuf_LIBRARIES=<protobuf-root-dir>/build-vs2017/install/lib/libprotobuf.lib -DProtobuf_PROTOC_EXECUTABLE=<protobuf-root-dir>/build-vs2017/install/bin/protoc.exe -DNCNN_VULKAN=OFF ..
    nmake
    nmake install
    #pick build-vs2017/install folder for further usage
  
__2.4 VS2017 settings__  
属性管理器-x64-Microsoft.cpp.64.user-右键属性  
+ 通用-vc++  
包含目录加入 ncnn protobuf opencv 的include文件夹   
库目录中加入 ncnn protobuf opencv 的lib文件夹  
+ 链接器-输入  
添加依赖项 `opencv_world420d.lib、ncnn.lib、libprotobuf.lib`  

## 3.use ncnn/examples  
copy `examples` folder to other directory  
modify CMakelist.txt  
```
cmake_minimum_required(VERSION 2.8.10)
project(ncnn-examples)
set(OpenCV_DIR "E:/VS/opencv/build")

find_package(OpenCV  QUIET COMPONENTS core highgui imgproc imgcodecs)
if(NOT OpenCV_FOUND)
    find_package(OpenCV QUIET COMPONENTS core highgui imgproc)
endif()

# add source and library of ncnn
include_directories(E:/VS/ncnn-master/src)
include_directories(E:/VS/ncnn-master/build-vs2017/src)
link_directories(E:/VS/ncnn-master/build-vs2017/install/lib)

if(OpenCV_FOUND)
    #add_executable(squeezenet squeezenet.cpp)
	#target_link_libraries(squeezenet ncnn ${OpenCV_LIBS})
    add_executable(simplepose simplepose.cpp)
	target_link_libraries(simplepose ncnn ${OpenCV_LIBS})
else()
    message(WARNING "OpenCV not found, examples won't be built")
endif()
```
  
At last  
```
>cmake  
>mkdir build
>cd build
>nmake
```
