#==================================================================
# DualSPHysics GPU/CPU v5.2.245 12-09-2022
#==================================================================

# 3.17 and 3.18 did a major update of CUDA support in CMake
cmake_minimum_required(VERSION 3.18)

project(DualSPHysics VERSION 5.2.245 LANGUAGES CXX)

# Precompiled libraries for Linux+GCC and Windows+MSVC are distributed with the source
# In these cases, default to using them.
# Without these, using options below with `default_precompiled` will need you to set the
set(default_precompiled_lib_dir NOTFOUND)
if (CMAKE_SYSTEM_NAME STREQUAL Linux)
  if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    set(default_precompiled_lib_dir ${CMAKE_CURRENT_SOURCE_DIR}/../lib/linux_gcc)
  endif()
elseif(CMAKE_SYSTEM_NAME STREQUAL Windows)
  if (MSVC)
    set(default_precompiled_lib_dir ${CMAKE_CURRENT_SOURCE_DIR}/../lib/vs2022)
  endif()
endif()

set(PRECOMPILED_LIB_DIR "${default_precompiled_lib_dir}"
  CACHE FILEPATH "Directory for precompiled dependencies")

if (PRECOMPILED_LIB_DIR)
  # By default enable the precompiled libraries
  set(default_precompiled ON)
else()
  # By default disable the precompiled libraries
  set(default_precompiled OFF)
endif()

option(ENABLE_CUDA "Enable CUDA version" ON)
#------------------------------------------------------------------
# Coupling options
#------------------------------------------------------------------
option(ENABLE_MOORDYN "Enable the MoorDyn+ library" ${default_precompiled})
option(ENABLE_CHRONO "Enable the Chrono Engine library" ${default_precompiled})
option(ENABLE_VTKLIB "Enable precompiled VTK wrapper" ${default_precompiled})
option(ENABLE_WAVEGEN "Enable precompiled wave generation" ${default_precompiled})
option(ENABLE_NUMEXLIB "Enable precompiled numexlib" ${default_precompiled})

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
  set(CMAKE_CXX_FLAGS_RELEASE "-use_fast_math -O3 -DNDEBUG")
endif()
set(CMAKE_CUDA_FLAGS_RELEASE "-use_fast_math -O3 -DNDEBUG")

function(get_supported_arch outvar)
  execute_process(
    COMMAND ${CMAKE_CUDA_COMPILER} --list-gpu-arch
    RESULT_VARIABLE result
    OUTPUT_VARIABLE arch_out
  )

  if (${result} EQUAL 0)
    # OK, can use compiler's list
    string(REPLACE "compute_" "" arch_out ${arch_out})
    string(REPLACE "\n" ";" arch_out ${arch_out})
    set(${outvar} "${arch_out}" PARENT_SCOPE)
  else()
    # Have to guess based on version
    if (CUDAToolkit_VERSION VERSION_LESS 9)
      set(arch "20;30;32;35;50;52;53")
    elseif (CUDAToolkit_VERSION VERSION_LESS 10)
      set(arch "30;32;35;50;52;53;60;61;62;70")
    elseif (CUDAToolkit_VERSION VERSION_LESS 11)
      set(arch "30;32;35;50;52;53;60;61;62;70;72;75")
    elseif (CUDAToolkit_VERSION VERSION_LESS 12)
      set(arch "30;32;35;50;52;53;60;61;62;70;72;75;80")
    else()
      set(arch "50;52;53;60;61;62;70;72;75;80;86;87;89;90")
    endif()
    set(${outvar} "${arch}" PARENT_SCOPE)
  endif()
endfunction()

if (ENABLE_CUDA)
  # First, figure out if arch has been set already, only do the guessing if not
  # Override this by setting CMAKE_CUDA_ARCHITECTURES or env{CUDAARCHS}
  if(NOT DEFINED CMAKE_CUDA_ARCHITECTURES)
    set(set_default_arch ON)
  else()
    # Make it appear in UI
    set(CMAKE_CUDA_ARCHITECTURES "${CMAKE_CUDA_ARCHITECTURES}" CACHE STRING "CUDA architectures")
  endif()

  enable_language(CUDA)
  set(CMAKE_CUDA_STANDARD 11)
  set(CMAKE_CUDA_STANDARD_REQUIRED ON)
  find_package(CUDAToolkit REQUIRED)

  if (set_default_arch)
    # No value for arch was specified by user. Enable all the wanted
    # ones that this version of CUDA supports.
    set(wanted_arch "20;30;35;37;50;52;61;70")
    message("No CUDA_ARCHITECTURES specified: will attempt to guess which of the following are supported: ${wanted_arch}")
    get_supported_arch(supported_arch)

    set(default_arch "")
    foreach(arch ${wanted_arch})
      if (arch IN_LIST supported_arch)
        list(APPEND default_arch ${arch})
      endif()
    endforeach()
    # Enabling CUDA sets a default for this. Iff we didn't have a
    # value set already, override the CMake default.
    set(CMAKE_CUDA_ARCHITECTURES "${default_arch}" CACHE STRING "CUDA architectures" FORCE)
  endif()

endif()

# Build the list of libraries
set(active_libraries "")

# Automate creating interface libraries for the dependencies.
# Signature:
# make_precompiled_lib(NAME <libname>
#                      GNU <list of libraries for Gnu>
#                      MSVC <list of libraries for MSVC>
#                      DEPS <list of other optional libs this one depends on>
# )
# Assumes an option of ENABLE_<LIBNAME> exists above
# and that the flag to disable in the code is DISABLE_<LIBNAME>
#
# Dependencies MUST already have been created AND be controlled by a similar flag
#
# If the option is enabled then create an interface library that actually links it.
#
# If it's disabled the interface library will simply define the disable macro.
macro(make_precompiled_lib)
  cmake_parse_arguments(_MPL "" "NAME" "GNU;MSVC;DEPS" ${ARGN})

  set(_mpl_flagname "ENABLE_${_MPL_NAME}")
  string(TOUPPER "${_mpl_flagname}" _mpl_flagname)
  set(_mpl_disable_def "DISABLE_${_MPL_NAME}")
  string(TOUPPER "${_mpl_disable_def}" _mpl_disable_def)

  add_library(${_MPL_NAME} INTERFACE)
  if (${_mpl_flagname})
    foreach(_mpl_dep ${_MPL_DEPS})
      set(_mpl_depflagname "ENABLE_${_mpl_dep}")
      string(TOUPPER "${_mpl_depflagname}" _mpl_depflagname)
      if (NOT ${_mpl_depflagname})
	message(FATAL_ERROR "Option ${_MPL_NAME} requires option ${_mpl_depflagname} be enabled")
      endif()
    endforeach()

    target_link_directories(${_MPL_NAME} INTERFACE "${PRECOMPILED_LIB_DIR}")
    if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
      target_link_libraries(${_MPL_NAME} INTERFACE "${_MPL_GNU}")
    elseif(MSVC)
      target_link_libraries(${_MPL_NAME} INTERFACE "${_MPL_MSVC}")
    endif()
    target_link_libraries(${_MPL_NAME} INTERFACE ${_MPL_DEPS})
  else()
    target_compile_definitions(${_MPL_NAME} INTERFACE ${_mpl_disable_def})
  endif()
  list(APPEND active_libraries ${_MPL_NAME})
endmacro()

# Apply above to the shipped libraries
make_precompiled_lib(NAME vtklib
  GNU jvtklib_64
  MSVC LibJVtkLib_x64_v143_Release
)
make_precompiled_lib(NAME moordyn
  GNU dsphmoordyn_64
  MSVC LibDSphMoorDyn_x64_v143_Release
  DEPS vtklib
)
make_precompiled_lib(NAME chrono
  GNU ChronoEngine dsphchrono
  MSVC dsphchrono
)
make_precompiled_lib(NAME numexlib
  GNU jnumexlib_64
  MSVC LibJNumexLib_x64_v143_Release
)
make_precompiled_lib(NAME wavegen
  GNU jwavegen_64
  MSVC LibJWaveGen_x64_v143_Release
  DEPS numexlib vtklib
)

#------------------------------------------------------------------
# Source files
#------------------------------------------------------------------

# CPU Objects
set(OBJXML JXml.cpp tinystr.cpp tinyxml.cpp tinyxmlerror.cpp tinyxmlparser.cpp)
set(OBJSPHMOTION JMotion.cpp JMotionList.cpp JMotionMov.cpp JMotionObj.cpp JMotionPos.cpp JDsMotion.cpp)
set(OBCOMMON Functions.cpp FunGeo3d.cpp FunSphKernelsCfg.cpp JAppInfo.cpp JBinaryData.cpp JCfgRunBase.cpp JDataArrays.cpp JException.cpp JLinearValue.cpp JLog2.cpp JObject.cpp JOutputCsv.cpp JRadixSort.cpp JRangeFilter.cpp JReadDatafile.cpp JSaveCsv2.cpp JTimeControl.cpp randomc.cpp)
set(OBCOMMONDSPH JDsphConfig.cpp JDsPips.cpp JPartDataBi4.cpp JPartDataHead.cpp JPartFloatBi4.cpp JPartOutBi4Save.cpp JCaseCtes.cpp JCaseEParms.cpp JCaseParts.cpp JCaseProperties.cpp JCaseUserVars.cpp JCaseVtkOut.cpp)
set(OBSPH JArraysCpu.cpp JCellDivCpu.cpp JSphCfgRun.cpp JComputeMotionRef.cpp JDsDcell.cpp JDsDamping.cpp JDsExtraData.cpp JDsGaugeItem.cpp JDsGaugeSystem.cpp JDsPartsOut.cpp JDsSaveDt.cpp JSphShifting.cpp JSph.cpp JDsAccInput.cpp JSphCpu.cpp JDsInitialize.cpp JFtMotionSave.cpp JSphMk.cpp JDsPartsInit.cpp JDsFixedDt.cpp JDsViscoInput.cpp JDsOutputTime.cpp JDsTimers.cpp JWaveSpectrumGpu.cpp main.cpp)
set(OBSPHSINGLE JCellDivCpuSingle.cpp JPartsLoad4.cpp JSphCpuSingle.cpp)

# GPU Objects
set(OBCOMMONGPU FunctionsCuda.cpp JObjectGpu.cpp)
set(OBSPHGPU JArraysGpu.cpp JDebugSphGpu.cpp JCellDivGpu.cpp JSphGpu.cpp JDsGpuInfo.cpp)
set(OBSPHSINGLEGPU JCellDivGpuSingle.cpp JSphGpuSingle.cpp)
set(OBCUDA JCellDivGpu_ker.cu JCellDivGpuSingle_ker.cu JDsPips_ker.cu JDsGauge_ker.cu JReduSum_ker.cu JSphShifting_ker.cu JDsAccInput_ker.cu JSphGpu_ker.cu JSphGpuSimple_ker.cu JWaveOrder2_ker.cu)

# Other Objects
set(OBWAVERZ JMLPistonsGpu.cpp JRelaxZonesGpu.cpp)
set(OBWAVERZCUDA JRelaxZone_ker.cu)
set(OBCHRONO JChronoObjects.cpp)
set(OBMOORDYN JDsMooredFloatings.cpp JDsFtForcePoints.cpp)
set(OBINOUT JSphCpu_InOut.cpp JSphCpuSingle_InOut.cpp JSphInOut.cpp JSphInOutZone.cpp JSphInOutGridData.cpp JSphInOutPoints.cpp JSphInOutVel.cpp JSphInOutVelAwas.cpp JSphInOutZsurf.cpp JSimpleNeigs.cpp)
set(OBINOUTGPU JSphGpuSingle_InOut.cpp)
set(OBMDBC JPartNormalData.cpp)

#------------------------------------------------------------------
# NVCC Flags
#------------------------------------------------------------------

if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
  set(PLATFORM "linux")
  set(plat_suffix linux64)
elseif(MSVC)
  message("Windows version")
  set(PLATFORM "windows")
  set(plat_suffix win64)
endif()

#--------------------------------------------
# OpenMP, if available
#---------------------------------------------
find_package(OpenMP)
if (OpenMP_FOUND)
  list(APPEND active_libraries OpenMP::OpenMP_CXX)
endif()

#------------------------------------------------------------------
# Binaries
#------------------------------------------------------------------
set (CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/../../bin/${PLATFORM}) # Generic output directory
set (CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})         # Release output directory
set (CMAKE_RUNTIME_OUTPUT_DIRECTORY_DEBUG ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})           # Debug output directory

# Create vars for the CPU and GPU executables
set(version "${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}")
set(cpu_exe "DualSPHysics${version}CPU_${plat_suffix}")
set(gpu_exe "DualSPHysics${version}_${plat_suffix}")

add_executable(${cpu_exe} ${OBJXML} ${OBJSPHMOTION} ${OBCOMMON} ${OBCOMMONDSPH} ${OBSPH} ${OBSPHSINGLE} ${OBWAVERZ} ${OBCHRONO} ${OBMOORDYN} ${OBINOUT} ${OBMDBC})
target_link_libraries(${cpu_exe} PUBLIC ${active_libraries})
install(TARGETS	${cpu_exe} DESTINATION ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})

if (ENABLE_CUDA)
  add_executable(${gpu_exe} ${OBJXML} ${OBJSPHMOTION} ${OBCOMMON} ${OBCOMMONDSPH} ${OBSPH} ${OBSPHSINGLE} ${OBCOMMONGPU} ${OBSPHGPU} ${OBSPHSINGLEGPU} ${OBCUDA} ${OBWAVERZ} ${OBWAVERZCUDA} ${OBCHRONO} ${OBMOORDYN} ${OBINOUT} ${OBINOUTGPU} ${OBMDBC})
  target_compile_definitions(${gpu_exe} PUBLIC _WITHGPU)
  target_link_libraries(${gpu_exe} PUBLIC ${active_libraries})
  target_link_libraries(${gpu_exe} PUBLIC CUDA::cudart)
  install(TARGETS ${gpu_exe} DESTINATION ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})
endif()

foreach(OUTPUTCONFIG ${CMAKE_CONFIGURATION_TYPES} )
  string(TOUPPER ${OUTPUTCONFIG} OUTPUTCONFIG )
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_${OUTPUTCONFIG} ${CMAKE_RUNTIME_OUTPUT_DIRECTORY} )
endforeach(OUTPUTCONFIG CMAKE_CONFIGURATION_TYPES )


#------------------------------------------------------------------
# Linker flags
#------------------------------------------------------------------
if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
  
  if (CUDA_FOUND)
    set_target_properties(DualSPHysics5.2_linux64 PROPERTIES COMPILE_FLAGS "-use_fast_math -O3 -fPIC -std=c++0x -Wno-deprecated-gpu-targets")
  endif(CUDA_FOUND)
  
elseif(MSVC)
  
  SET(CUDA_PROPAGATE_HOST_FLAGS OFF CACHE BOOL "Propagate C/CXX Flags and friends to the host compiler in NVCC via -Xompile  " FORCE)
  
  foreach(CPP_FLAGS CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELWITHDEBINFO)
    if(${CPP_FLAGS} MATCHES "/MD")
      string(REGEX REPLACE "/MD" "/MT" ${CPP_FLAGS} "${${CPP_FLAGS}}")
    endif(${CPP_FLAGS} MATCHES "/MD")
  endforeach(CPP_FLAGS)
endif()
