cmake_minimum_required (VERSION 3.6)
project (UNIPAR)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_C_STANDARD 11)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH}) # To allow CMake to locate our Find*.cmake files
message("${CMAKE_BINARY_DIR}")
message("${CMAKE_CXX_COMPILER}")

find_package(TBB REQUIRED)
if (TBB_FOUND)
    message("${TBB_VERSION}, ${TBB_INCLUDE_DIRS}, ${TBB_LIBRARIES}")
    include_directories(${TBB_INCLUDE_DIRS})
    link_libraries(${TBB_LIBRARIES})
endif ()
add_library(tbbsort STATIC tbbsort.cpp)
target_link_libraries(tbbsort PRIVATE ${TBB_LIBRARIES})

#find_package(MPI REQUIRED)
if (MPI_FOUND)
	message("${MPI_VERSION}, ${MPI_INCLUDE_DIRS}, ${MPI_LIBRARIES}")
	include_directories(${MPI_INCLUDE_DIRS})
else ()
	set(MPI_LIBRARIES /home/sqiuac/usr/local/lib/libmpi.so)
	set(MPI_INCLUDE_DIRS /home/sqiuac/usr/local/include)
endif ()

add_library(distribute SHARED distribute.c pio.c)
target_link_libraries(distribute PRIVATE ${MPI_LIBRARIES})

add_subdirectory(hash)

find_package(OpenMP REQUIRED)
if (OPENMP_FOUND)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${OpenMP_EXE_LINKER_FLAGS}")
else ()
    # icpc
    if ("${CMAKE_CXX_COMPILER_ID}" MATCHES "Intel")
        message(icpc)
        set(CMAKE_C_FLAGS "${CMAKE_CXX_FLAGS} -qopenmp")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -qopenmp")
    elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
        message(g++)
        set(CMAKE_C_FLAGS "${CMAKE_CXX_FLAGS} -fopenmp")
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fopenmp")
    endif ()
endif ()

set(SOURCE_FILES 
	bitkmer.c comm.c contig.c dbgraph.c 
	io.c malloc.c malloc.cuh msp.c preprocess.c
	share.c main.c)

add_executable(unipar ${SOURCE_FILES})

find_package("CUDA" REQUIRED)	
    macro(cuda_add_compile_options options)
        list(APPEND CUDA_NVCC_FLAGS "${options}")
    endmacro()
    cuda_add_compile_options("-gencode arch=compute_35,code=sm_35")
    cuda_add_compile_options("-std=c++11 -O3 -g -lineinfo")
    cuda_add_compile_options("-Xcompiler ${OpenMP_CXX_FLAGS} -Xcompiler -march=core-avx2")
    
cuda_add_library(culib SHARED all2all.cu comm.cu contig.cu dbgraph.cu msp.cu preprocess.cu share.cu)

link_directories(${CMAKE_BINARY_DIR}/hash) 
target_link_libraries(unipar /home/sqiuac/.linuxbrew/lib/librt.so.1)   
target_link_libraries(unipar ${CMAKE_THREAD_LIBS_INIT})
target_link_libraries(unipar tbbsort)
target_link_libraries(unipar distribute)
target_link_libraries(unipar hash)
target_link_libraries(unipar culib)

set_target_properties(unipar PROPERTIES CUDA_SEPARABLE_COMPILATION ON)
