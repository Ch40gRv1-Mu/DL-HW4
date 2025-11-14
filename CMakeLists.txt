cmake_minimum_required(VERSION 3.5)
project(needle C CXX)
if(POLICY CMP0146)
  cmake_policy(SET CMP0146 OLD)
endif()

if(NOT CMAKE_CUDA_HOST_COMPILER)
  set(CMAKE_CUDA_HOST_COMPILER ${CMAKE_CXX_COMPILER})
endif()
set(CUDA_HOST_COMPILER ${CMAKE_CXX_COMPILER} CACHE FILEPATH "CUDA host compiler" FORCE)
# find correct version of Python
execute_process(COMMAND python3-config --prefix
  OUTPUT_VARIABLE Python_ROOT_DIR)
find_package(Python COMPONENTS Development Interpreter REQUIRED)
include_directories(${Python_INCLUDE_DIRS})

# find pybind
execute_process(COMMAND python3 -m pybind11 --cmakedir
  RESULT_VARIABLE __pybind_exit_code
  OUTPUT_VARIABLE __pybind_path
  OUTPUT_STRIP_TRAILING_WHITESPACE)
find_package(pybind11 PATHS ${__pybind_path})


set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(CMAKE_CUDA_STANDARD 17)
set(CMAKE_CUDA_STANDARD_REQUIRED ON)
set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} --expt-relaxed-constexpr --expt-extended-lambda")

if(NOT MSVC)
  set(CMAKE_CXX_FLAGS "-O2 -march=native ${CMAKE_CXX_FLAGS}")
else()
  set(CMAKE_CXX_FLAGS "/O2 /arch:AVX2 ${CMAKE_CXX_FLAGS}")
endif()

include_directories(SYSTEM ${pybind11_INCLUDE_DIRS})
list(APPEND LINKER_LIBS ${pybind11_LIBRARIES})


###################
### CPU BACKEND ###
###################
add_library(ndarray_backend_cpu MODULE src/ndarray_backend_cpu.cc)
target_link_libraries(ndarray_backend_cpu PUBLIC ${LINKER_LIBS})
pybind11_extension(ndarray_backend_cpu)
pybind11_strip(ndarray_backend_cpu)


# directly output to ffi folder
set_target_properties(ndarray_backend_cpu
  PROPERTIES
  LIBRARY_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/python/needle/backend_ndarray
  CXX_VISIBILITY_PRESET "hidden"
)

if(${CMAKE_SYSTEM_NAME} MATCHES "Darwin")
  set_property(TARGET ndarray_backend_cpu PROPERTY LINK_OPTIONS -undefined dynamic_lookup)
endif()



####################
### CUDA BACKEND ###
####################
find_package(CUDA)
if(CUDA_FOUND)
  message(STATUS "Found cuda, building cuda backend")

  include_directories(SYSTEM ${CUDA_INCLUDE_DIRS})
  list(APPEND LINKER_LIBS ${CUDA_CUDART_LIBRARY})


  # set arch flags properly
  CUDA_ADD_LIBRARY(ndarray_backend_cuda MODULE src/ndarray_backend_cuda.cu OPTIONS ${ARCH_FLAGS})

  target_link_libraries(ndarray_backend_cuda ${LINKER_LIBS})
  pybind11_extension(ndarray_backend_cuda)
  pybind11_strip(ndarray_backend_cuda)

  # directly output to ffi folder
  set_target_properties(ndarray_backend_cuda
    PROPERTIES
    LIBRARY_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/python/needle/backend_ndarray
    CXX_VISIBILITY_PRESET "hidden"
    CUDA_VISIBILITY_PRESET "hidden"
)

endif()
