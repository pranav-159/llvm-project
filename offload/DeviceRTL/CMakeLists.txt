set(LIBOMPTARGET_BUILD_DEVICERTL_BCLIB TRUE CACHE BOOL
  "Can be set to false to disable building this library.")

if (NOT LIBOMPTARGET_BUILD_DEVICERTL_BCLIB)
  message(STATUS "Not building DeviceRTL: Disabled by LIBOMPTARGET_BUILD_DEVICERTL_BCLIB")
  return()
endif()

# Check to ensure the host system is a supported host architecture.
if(NOT ${CMAKE_SIZEOF_VOID_P} EQUAL "8")
  message(STATUS "Not building DeviceRTL: Runtime does not support 32-bit hosts")
  return()
endif()

if (LLVM_DIR)
  # Builds that use pre-installed LLVM have LLVM_DIR set.
  # A standalone or LLVM_ENABLE_RUNTIMES=openmp build takes this route
  find_program(CLANG_TOOL clang PATHS ${LLVM_TOOLS_BINARY_DIR} NO_DEFAULT_PATH)
  find_program(PACKAGER_TOOL clang-offload-packager PATHS ${LLVM_TOOLS_BINARY_DIR} NO_DEFAULT_PATH)
  find_program(LINK_TOOL llvm-link PATHS ${LLVM_TOOLS_BINARY_DIR} NO_DEFAULT_PATH)
  find_program(OPT_TOOL opt PATHS ${LLVM_TOOLS_BINARY_DIR} NO_DEFAULT_PATH)
  if ((NOT CLANG_TOOL) OR (NOT LINK_TOOL) OR (NOT OPT_TOOL) OR (NOT PACKAGER_TOOL))
    message(STATUS "Not building DeviceRTL. Missing clang: ${CLANG_TOOL}, llvm-link: ${LINK_TOOL}, opt: ${OPT_TOOL}, or clang-offload-packager: ${PACKAGER_TOOL}")
    return()
  else()
    message(STATUS "Building DeviceRTL. Using clang: ${CLANG_TOOL}, llvm-link: ${LINK_TOOL} and opt: ${OPT_TOOL}")
  endif()
elseif (LLVM_TOOL_CLANG_BUILD AND NOT CMAKE_CROSSCOMPILING AND NOT OPENMP_STANDALONE_BUILD)
  # LLVM in-tree builds may use CMake target names to discover the tools.
  # A LLVM_ENABLE_PROJECTS=openmp build takes this route
  set(CLANG_TOOL $<TARGET_FILE:clang>)
  set(PACKAGER_TOOL $<TARGET_FILE:clang-offload-packager>)
  set(LINK_TOOL $<TARGET_FILE:llvm-link>)
  set(OPT_TOOL $<TARGET_FILE:opt>)
  message(STATUS "Building DeviceRTL. Using clang from in-tree build")
else()
  message(STATUS "Not building DeviceRTL. No appropriate clang found")
  return()
endif()

set(devicertl_base_directory ${CMAKE_CURRENT_SOURCE_DIR})
set(include_directory ${devicertl_base_directory}/include)
set(source_directory ${devicertl_base_directory}/src)

set(all_amdgpu_architectures "gfx700;gfx701;gfx801;gfx803;gfx900;gfx902;gfx906"
                             "gfx908;gfx90a;gfx90c;gfx940;gfx941;gfx942;gfx950;gfx1010"
                             "gfx1012;gfx1030;gfx1031;gfx1032;gfx1033;gfx1034;gfx1035"
                             "gfx1036;gfx1100;gfx1101;gfx1102;gfx1103;gfx1150"
                             "gfx1151;gfx1152;gfx1153")
set(all_nvptx_architectures "sm_35;sm_37;sm_50;sm_52;sm_53;sm_60;sm_61;sm_62"
                            "sm_70;sm_72;sm_75;sm_80;sm_86;sm_87;sm_89;sm_90")
set(all_gpu_architectures
    "${all_amdgpu_architectures};${all_nvptx_architectures}")

set(LIBOMPTARGET_DEVICE_ARCHITECTURES "all" CACHE STRING
    "List of device architectures to be used to compile the OpenMP DeviceRTL.")

if(LIBOMPTARGET_DEVICE_ARCHITECTURES STREQUAL "all")
  set(LIBOMPTARGET_DEVICE_ARCHITECTURES ${all_gpu_architectures})
elseif(LIBOMPTARGET_DEVICE_ARCHITECTURES STREQUAL "amdgpu")
  set(LIBOMPTARGET_DEVICE_ARCHITECTURES ${all_amdgpu_architectures})
elseif(LIBOMPTARGET_DEVICE_ARCHITECTURES STREQUAL "nvptx")
  set(LIBOMPTARGET_DEVICE_ARCHITECTURES ${all_nvptx_architectures})
elseif(LIBOMPTARGET_DEVICE_ARCHITECTURES STREQUAL "auto" OR
       LIBOMPTARGET_DEVICE_ARCHITECTURES STREQUAL "native")
  if(NOT LIBOMPTARGET_NVPTX_ARCH AND NOT LIBOMPTARGET_AMDGPU_ARCH)
    message(FATAL_ERROR
      "Could not find 'amdgpu-arch' and 'nvptx-arch' tools required for 'auto'")
  elseif(NOT LIBOMPTARGET_FOUND_NVIDIA_GPU AND NOT LIBOMPTARGET_FOUND_AMDGPU_GPU)
    message(FATAL_ERROR "No AMD or NVIDIA GPU found on the system when using 'auto'")
  endif()
  set(LIBOMPTARGET_DEVICE_ARCHITECTURES
      "${LIBOMPTARGET_NVPTX_DETECTED_ARCH_LIST};${LIBOMPTARGET_AMDGPU_DETECTED_ARCH_LIST}")
endif()
list(REMOVE_DUPLICATES LIBOMPTARGET_DEVICE_ARCHITECTURES)

set(include_files
  ${include_directory}/Allocator.h
  ${include_directory}/Configuration.h
  ${include_directory}/Debug.h
  ${include_directory}/Interface.h
  ${include_directory}/LibC.h
  ${include_directory}/Mapping.h
  ${include_directory}/Profiling.h
  ${include_directory}/State.h
  ${include_directory}/Synchronization.h
  ${include_directory}/DeviceTypes.h
  ${include_directory}/DeviceUtils.h
  ${include_directory}/Workshare.h
)

set(src_files
  ${source_directory}/Allocator.cpp
  ${source_directory}/Configuration.cpp
  ${source_directory}/Debug.cpp
  ${source_directory}/Kernel.cpp
  ${source_directory}/LibC.cpp
  ${source_directory}/Mapping.cpp
  ${source_directory}/Misc.cpp
  ${source_directory}/Parallelism.cpp
  ${source_directory}/Profiling.cpp
  ${source_directory}/Reduction.cpp
  ${source_directory}/State.cpp
  ${source_directory}/Synchronization.cpp
  ${source_directory}/Tasking.cpp
  ${source_directory}/DeviceUtils.cpp
  ${source_directory}/Workshare.cpp
)

# We disable the slp vectorizer during the runtime optimization to avoid
# vectorized accesses to the shared state. Generally, those are "good" but
# the optimizer pipeline (esp. Attributor) does not fully support vectorized
# instructions yet and we end up missing out on way more important constant
# propagation. That said, we will run the vectorizer again after the runtime
# has been linked into the user program.
set(clang_opt_flags -O3 -mllvm -openmp-opt-disable -DSHARED_SCRATCHPAD_SIZE=512 -mllvm -vectorize-slp=false )
set(link_opt_flags  -O3        -openmp-opt-disable -attributor-enable=module -vectorize-slp=false )
set(link_export_flag -passes=internalize -internalize-public-api-file=${source_directory}/exports)

# If the user built with the GPU C library enabled we will use that instead.
if(${LIBOMPTARGET_GPU_LIBC_SUPPORT})
  list(APPEND clang_opt_flags -DOMPTARGET_HAS_LIBC)
endif()

# Prepend -I to each list element
set (LIBOMPTARGET_LLVM_INCLUDE_DIRS_DEVICERTL "${LIBOMPTARGET_LLVM_INCLUDE_DIRS}")
list(TRANSFORM LIBOMPTARGET_LLVM_INCLUDE_DIRS_DEVICERTL PREPEND "-I")

# Set flags for LLVM Bitcode compilation.
set(bc_flags -c -foffload-lto -std=c++17 -fvisibility=hidden
              ${clang_opt_flags} --offload-device-only
             -nocudalib -nogpulib -nogpuinc -nostdlibinc
             -fopenmp -fopenmp-cuda-mode
             -Wno-unknown-cuda-version
             -DOMPTARGET_DEVICE_RUNTIME
             -I${include_directory}
             -I${devicertl_base_directory}/../include
             -I${devicertl_base_directory}/../../libc
             ${LIBOMPTARGET_LLVM_INCLUDE_DIRS_DEVICERTL}
)

# first create an object target
add_library(omptarget.devicertl.all_objs OBJECT IMPORTED)
function(compileDeviceRTLLibrary target_cpu target_name target_triple)
  set(target_bc_flags ${ARGN})

  set(bc_files "")
  foreach(src ${src_files})
    get_filename_component(infile ${src} ABSOLUTE)
    get_filename_component(outfile ${src} NAME)
    set(outfile "${outfile}-${target_cpu}.bc")
    set(depfile "${outfile}.d")

    add_custom_command(OUTPUT ${outfile}
      COMMAND ${CLANG_TOOL}
      ${bc_flags}
      --offload-arch=${target_cpu}
      ${target_bc_flags}
      -MD -MF ${depfile}
      ${infile} -o ${outfile}
      DEPENDS ${infile}
      DEPFILE ${depfile}
      COMMENT "Building LLVM bitcode ${outfile}"
      VERBATIM
    )
    if(TARGET clang)
      # Add a file-level dependency to ensure that clang is up-to-date.
      # By default, add_custom_command only builds clang if the
      # executable is missing.
      add_custom_command(OUTPUT ${outfile}
        DEPENDS clang
        APPEND
      )
    endif()
    set_property(DIRECTORY APPEND PROPERTY ADDITIONAL_MAKE_CLEAN_FILES ${outfile})

    list(APPEND bc_files ${outfile})
  endforeach()

  set(bclib_name "libomptarget-${target_name}-${target_cpu}.bc")

  # Link to a bitcode library.
  add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/linked_${bclib_name}
      COMMAND ${LINK_TOOL}
        -o ${CMAKE_CURRENT_BINARY_DIR}/linked_${bclib_name} ${bc_files}
      DEPENDS ${bc_files}
      COMMENT "Linking LLVM bitcode ${bclib_name}"
  )

  if(TARGET llvm-link)
    add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/linked_${bclib_name}
      DEPENDS llvm-link
      APPEND)
  endif()

  add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/internalized_${bclib_name}
      COMMAND ${OPT_TOOL} ${link_export_flag} ${CMAKE_CURRENT_BINARY_DIR}/linked_${bclib_name}
                      -o ${CMAKE_CURRENT_BINARY_DIR}/internalized_${bclib_name}
      DEPENDS ${source_directory}/exports ${CMAKE_CURRENT_BINARY_DIR}/linked_${bclib_name}
      COMMENT "Internalizing LLVM bitcode ${bclib_name}"
  )
  if(TARGET opt)
    add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/internalized_${bclib_name}
      DEPENDS opt
      APPEND)
  endif()

  add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${bclib_name}
      COMMAND ${OPT_TOOL} ${link_opt_flags} ${CMAKE_CURRENT_BINARY_DIR}/internalized_${bclib_name}
                      -o ${CMAKE_CURRENT_BINARY_DIR}/${bclib_name}
      DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/internalized_${bclib_name}
      COMMENT "Optimizing LLVM bitcode ${bclib_name}"
  )
  if(TARGET opt)
    add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${bclib_name}
      DEPENDS opt
      APPEND)
  endif()

  set(bclib_target_name "omptarget-${target_name}-${target_cpu}-bc")
  add_custom_target(${bclib_target_name} DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/${bclib_name})

  # Copy library to destination.
  add_custom_command(TARGET ${bclib_target_name} POST_BUILD
                    COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_BINARY_DIR}/${bclib_name}
                    ${LIBOMPTARGET_LIBRARY_DIR})
  add_dependencies(omptarget.devicertl.${target_name} ${bclib_target_name})

  set_property(DIRECTORY APPEND PROPERTY ADDITIONAL_MAKE_CLEAN_FILES ${bclib_name} ${LIBOMPTARGET_LIBRARY_DIR}/${bclib_name})

  # Install bitcode library under the lib destination folder.
  install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${bclib_name} DESTINATION "${OFFLOAD_INSTALL_LIBDIR}")

  set(target_feature "")
  if("${target_triple}" STREQUAL "nvptx64-nvidia-cuda")
    set(target_feature "feature=+ptx63")
  endif()

  # Package the bitcode in the bitcode and embed it in an ELF for the static library
  add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/packaged_${bclib_name}
      COMMAND ${PACKAGER_TOOL} -o ${CMAKE_CURRENT_BINARY_DIR}/packaged_${bclib_name}
        "--image=file=${CMAKE_CURRENT_BINARY_DIR}/${bclib_name},${target_feature},triple=${target_triple},arch=${target_cpu},kind=openmp"
      DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/${bclib_name}
      COMMENT "Packaging LLVM offloading binary ${bclib_name}.out"
  )
  if(TARGET clang-offload-packager)
    add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/packaged_${bclib_name}
      DEPENDS clang-offload-packager
      APPEND)
  endif()

  set(output_name "${CMAKE_CURRENT_BINARY_DIR}/devicertl-${target_name}-${target_cpu}.o")
  add_custom_command(OUTPUT ${output_name}
    COMMAND ${CLANG_TOOL} --std=c++17 -c -nostdlib
            -Xclang -fembed-offload-object=${CMAKE_CURRENT_BINARY_DIR}/packaged_${bclib_name}
            -o ${output_name}
            ${source_directory}/Stub.cpp
    DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/packaged_${bclib_name} ${source_directory}/Stub.cpp
    COMMENT "Embedding LLVM offloading binary in devicertl-${target_name}-${target_cpu}.o"
    VERBATIM
  )
  if(TARGET clang)
    add_custom_command(OUTPUT ${output_name}
      DEPENDS clang
      APPEND)
  endif()

  set_property(DIRECTORY APPEND PROPERTY ADDITIONAL_MAKE_CLEAN_FILES ${output_name})
  set_property(TARGET omptarget.devicertl.all_objs APPEND PROPERTY IMPORTED_OBJECTS ${output_name})

  if (CMAKE_EXPORT_COMPILE_COMMANDS)
    set(ide_target_name omptarget-ide-${target_name}-${target_cpu})
    add_library(${ide_target_name} STATIC EXCLUDE_FROM_ALL ${src_files})
    target_compile_options(${ide_target_name} PRIVATE
      -fopenmp --offload-arch=${target_cpu} -fopenmp-cuda-mode
      -mllvm -openmp-opt-disable
      -foffload-lto -fvisibility=hidden --offload-device-only
      -nocudalib -nogpulib -nogpuinc -nostdlibinc -Wno-unknown-cuda-version
    )
    target_compile_definitions(${ide_target_name} PRIVATE SHARED_SCRATCHPAD_SIZE=512)
    target_include_directories(${ide_target_name} PRIVATE
      ${include_directory}
      ${devicertl_base_directory}/../../libc
      ${devicertl_base_directory}/../include
      ${LIBOMPTARGET_LLVM_INCLUDE_DIRS}
    )
    install(TARGETS ${ide_target_name} EXCLUDE_FROM_ALL)
  endif()
endfunction()

# Generate a Bitcode library for all the gpu architectures the user requested.
add_custom_target(omptarget.devicertl.nvptx)
add_custom_target(omptarget.devicertl.amdgpu)
foreach(gpu_arch ${LIBOMPTARGET_DEVICE_ARCHITECTURES})
  if("${gpu_arch}" IN_LIST all_amdgpu_architectures)
    compileDeviceRTLLibrary(${gpu_arch} amdgpu amdgcn-amd-amdhsa -Xclang -mcode-object-version=none)
  elseif("${gpu_arch}" IN_LIST all_nvptx_architectures)
    compileDeviceRTLLibrary(${gpu_arch} nvptx nvptx64-nvidia-cuda --cuda-feature=+ptx63)
  else()
    message(FATAL_ERROR "Unknown GPU architecture '${gpu_arch}'")
  endif()
endforeach()

# Archive all the object files generated above into a static library
add_library(omptarget.devicertl STATIC)
set_target_properties(omptarget.devicertl PROPERTIES
  ARCHIVE_OUTPUT_DIRECTORY "${LIBOMPTARGET_LLVM_LIBRARY_INTDIR}"
  LINKER_LANGUAGE CXX
)
target_link_libraries(omptarget.devicertl PRIVATE omptarget.devicertl.all_objs)

install(TARGETS omptarget.devicertl ARCHIVE DESTINATION ${OFFLOAD_INSTALL_LIBDIR})
