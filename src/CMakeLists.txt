
#========================================================================
# Author: Richard Brown, Kris Thielemans
# Copyright 2016 - 2020 University College London
# Copyright 2016 - 2020 Science Technology Facilities Council
#
#  Licensed under the Apache License, Version 2.0 (the "License");
#  you may not use this file except in compliance with the License.
#  You may obtain a copy of the License at
#
#         http://www.apache.org/licenses/LICENSE-2.0.txt
#
#  Unless required by applicable law or agreed to in writing, software
#  distributed under the License is distributed on an "AS IS" BASIS,
#  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#  See the License for the specific language governing permissions and
#  limitations under the License.
#
#=========================================================================

ADD_SUBDIRECTORY(iUtilities)
#ADD_SUBDIRECTORY(common)


##########################################################################
#                              Gadgetron                                 #
##########################################################################
option(DISABLE_Gadgetron "Disable building the SIRF interface to Gadgetron" OFF)
if (DISABLE_Gadgetron)
  message(STATUS "Gadgetron support disabled.")
else()
  find_package(ISMRMRD 1.4 REQUIRED)
  # Add ISMRMRD to search path for FFTW3
  set(CMAKE_MODULE_PATH "${CMAKE_MODULE_PATH};${ISMRMRD_DIR}")
  find_package(FFTW3 COMPONENTS single REQUIRED)
  ADD_SUBDIRECTORY(xGadgetron)
  set(SIRF_BUILT_WITH_ISMRMRD TRUE PARENT_SCOPE)
  set(ISMRMRD_VERSION ${ISMRMRD_VERSION} PARENT_SCOPE)
endif()



##########################################################################
#                             Registration                               #
##########################################################################
option(DISABLE_Registration "Disable building the SIRF registration package" OFF)
if (DISABLE_Registration)
  message(STATUS "Registration support disabled.")
  set(SPM_BOOL_STR "0" PARENT_SCOPE)
  set(SIRF_BUILT_WITH_REGISTRATION False PARENT_SCOPE)
else()
  FIND_PACKAGE(NIFTYREG 1.5.61 REQUIRED)
  # NIFTYREG
  FOREACH(NR_lib ${NIFTYREG_LIBRARIES})
    find_library(${NR_lib}_full_path "${NR_lib}" "${NIFTYREG_LIBRARY_DIRS}")
    if(NOT ${NR_lib}_full_path)
      message(FATAL_ERROR "${NR_lib} not found")
    endif()
    SET(NR_libs_full_path "${NR_libs_full_path};${${NR_lib}_full_path}")
  ENDFOREACH()
  # If niftyreg was bulit with OpenMP
  if (NIFTYREG_BUILT_WITH_OPENMP)
    find_package(OpenMP REQUIRED)
    if (OpenMP_CXX_FOUND)
      SET(NR_libs_full_path "${NR_libs_full_path};OpenMP::OpenMP_CXX")
    endif()
  endif()
  # If niftyreg was bulit with CUDA
  if (NIFTYREG_BUILT_WITH_CUDA)
    find_package(CUDA REQUIRED)
    SET(NR_libs_full_path "${NR_libs_full_path};${CUDA_CUDA_LIBRARY};${CUDA_CUDART_LIBRARY}")
  endif()
  ADD_SUBDIRECTORY(Registration)
  set(SPM_BOOL_STR ${SPM_BOOL_STR} PARENT_SCOPE)
  set(SIRF_BUILT_WITH_REGISTRATION TRUE PARENT_SCOPE)
endif()



##########################################################################
#                                 STIR                                   #
##########################################################################
option(DISABLE_STIR "Disable building the SIRF interface to STIR" OFF)
if (DISABLE_STIR)
  message(STATUS "STIR support disabled.")
  set(NiftyPET_BOOL_STR "0" PARENT_SCOPE)
  set(Parallelproj_BOOL_STR "0" PARENT_SCOPE)
else()
  find_package(STIR REQUIRED)
  message(STATUS "STIR version found: ${STIR_VERSION}")
  if (STIR_VERSION  VERSION_LESS 4.1.0)
    # Note: we support both version 4 and 5, so cannot put the version in the find_package statement
    message(FATAL_ERROR "SIRF requires STIR version at least 4.1.0")
  endif()
  if (STIR_WITH_NiftyPET_PROJECTOR)
    set(NiftyPET_BOOL_STR "1")
  	message(STATUS "STIR was built with NiftyPET, GPU projectors will be enabled.")
    if(${CMAKE_VERSION} VERSION_LESS "3.12.0")
      add_definitions(-DSTIR_WITH_NiftyPET_PROJECTOR)
    else()
      add_compile_definitions(STIR_WITH_NiftyPET_PROJECTOR)
    endif()
  else()
    MESSAGE(STATUS "STIR not built with NiftyPET.")
    set(NiftyPET_BOOL_STR "0")
  endif()
  if (STIR_WITH_Parallelproj_PROJECTOR)
    set(Parallelproj_BOOL_STR "1")
  	message(STATUS "STIR was built with Parallelproj, parallelproj projectors will be enabled.")
  else()
    MESSAGE(STATUS "STIR not built with Parallelproj.")
    set(Parallelproj_BOOL_STR "0")
  endif()
  ADD_SUBDIRECTORY(xSTIR)
  set(SIRF_BUILT_WITH_STIR TRUE PARENT_SCOPE)
  set(STIR_VERSION ${STIR_VERSION} PARENT_SCOPE)
endif()
set(NiftyPET_BOOL_STR ${NiftyPET_BOOL_STR} PARENT_SCOPE)
set(Parallelproj_BOOL_STR ${Parallelproj_BOOL_STR} PARENT_SCOPE)


##########################################################################
#                             Synergistic                                #
##########################################################################
# Synergistic code currently only works if STIR (with ITK), ISMRMRD and Registration are built
option(DISABLE_Synergistic "Disable building the synergistic code" OFF)
if (DISABLE_Synergistic)
  message(STATUS "Synergistic code disabled.")
else()
  if ((NOT DISABLE_STIR) AND (NOT DISABLE_Gadgetron) AND (NOT DISABLE_Registration) AND "${STIR_BUILT_WITH_ITK}")
    MESSAGE(STATUS "Registration, ISMRMRD and STIR (with ITK) have been built. Building synergistic code.")
    ADD_SUBDIRECTORY(Synergistic)
  else()
    MESSAGE(STATUS "One or more of the following components are missing: Registration, ISMRMRD and STIR (with ITK). Synergistic code will not be built.")
    if (DISABLE_STIR)
      MESSAGE(STATUS "STIR missing.")
    elseif(NOT STIR_BUILT_WITH_ITK)
      MESSAGE(STATUS "STIR not built with ITK.")
    endif()
    if (DISABLE_Gadgetron)
      MESSAGE(STATUS "ISMRMRD missing.")
    endif()
    if (DISABLE_Registration)
      MESSAGE(STATUS "Registration missing.")
    endif()
  endif()
endif()

ADD_SUBDIRECTORY(common)
