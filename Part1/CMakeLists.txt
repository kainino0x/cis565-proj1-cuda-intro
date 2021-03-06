cmake_minimum_required(VERSION 3.0.0)

project(565simulator)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake")

#Find up and set up core dependency libs
find_package(GLUT)
find_package(OpenGL)
find_package(GLEW)

set(CORELIBS ${GLFW_LIBRARY} ${GLUT_LIBRARY} ${OPENGL_LIBRARY} ${GLEW_LIBRARY})

#OSX-specific hacks/fixes
if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
    #on OSX we have to explicitly set clang/clang++
    set (CMAKE_C_COMPILER clang)
    set (CMAKE_CXX_COMPILER clang++)

   	#Link IOKit because this is where we get GL stuff for OSX
   	set(IOKIT "-framework IOKit")
	set(CORELIBS ${CORELIBS} ${IOKIT})
	#Link against libstdc++ since CUDA doesn't support libc++ yet
	set(CUDA_NVCC_FLAGS "--compiler-options;-stdlib=libstdc++")
endif(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")

#Linux specific hacks/fixes
if(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
	set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -lX11 -lXxf86vm -lXrandr -lpthread -lXi")
endif()

#Compiler flag magic
if(${CMAKE_SYSTEM_NAME} MATCHES "Linux" OR ${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3 -mtune=native")
	set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -O3")
	set(CMAKE_CXX_FLAGS_DEBUG   "${CMAKE_CXX_FLAGS_DEBUG}   -O0 -g")
endif()
if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libstdc++")
endif()

#Crucial magic for CUDA linking
find_package(CUDA REQUIRED)
set(CUDA_ATTACH_VS_BUILD_RULE_TO_CUDA_FILE ON)
set(CUDA_SEPARABLE_COMPILATION ON)

#Force Visual Studio to link against MT versions of libs
if(MSVC)
	set(CompilerFlags
	        CMAKE_CXX_FLAGS
	        CMAKE_CXX_FLAGS_DEBUG
	        CMAKE_CXX_FLAGS_RELEASE
	        CMAKE_C_FLAGS
	        CMAKE_C_FLAGS_DEBUG
	        CMAKE_C_FLAGS_RELEASE
	        )
	foreach(CompilerFlag ${CompilerFlags})
	  string(REPLACE "/MD" "/MT" ${CompilerFlag} "${${CompilerFlag}}")
	endforeach()
	set(CUDA_NVCC_FLAGS_RELEASE ${CUDA_NVCC_FLAGS_RELEASE} "--compiler-options /MT; --linker-options /MT")
	set(CUDA_NVCC_FLAGS_DEBUG ${CUDA_NVCC_FLAGS_DEBUG} "--compiler-options /MT; --linker-options /MT")
endif()

#Add all source files. Headers don't need to be listed here since the compiler will find them;
#we just need the actual files being fed directly to the compiler
set(SOURCE_FILES ${SOURCE_FILES}
    "src/main.cpp"
    "src/kernel.cu"
    "src/glslUtility.cpp"
    )

cuda_add_executable(${CMAKE_PROJECT_NAME} ${SOURCE_FILES} OPTIONS -arch=sm_20)

target_link_libraries(${CMAKE_PROJECT_NAME} ${CORELIBS})

file(COPY shaders
    DESTINATION .)
