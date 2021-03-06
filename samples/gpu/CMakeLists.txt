SET(OPENCV_GPU_SAMPLES_REQUIRED_DEPS opencv_core opencv_flann opencv_imgproc opencv_highgui
                                     opencv_ml opencv_video opencv_objdetect opencv_features2d
                                     opencv_calib3d opencv_legacy opencv_contrib opencv_gpu
                                     opencv_superres)
ocv_check_dependencies(${OPENCV_GPU_SAMPLES_REQUIRED_DEPS})

if(BUILD_EXAMPLES AND OCV_DEPENDENCIES_FOUND)
  set(project "gpu")
  string(TOUPPER "${project}" project_upper)

  project("${project}_samples")

  ocv_include_modules(${OPENCV_GPU_SAMPLES_REQUIRED_DEPS})
  ocv_include_directories(
    "${OpenCV_SOURCE_DIR}/modules/gpu/src/nvidia"
    "${OpenCV_SOURCE_DIR}/modules/gpu/src/nvidia/core"
    )

  if(HAVE_opencv_nonfree)
    ocv_include_directories("${OpenCV_SOURCE_DIR}/modules/nonfree/include")
  endif()

  if(HAVE_CUDA)
    ocv_include_directories(${CUDA_INCLUDE_DIRS})
  endif()

  if(HAVE_OPENCL)
    ocv_include_directories("${OpenCV_SOURCE_DIR}/modules/ocl/include")
  endif()

  if(CMAKE_COMPILER_IS_GNUCXX AND NOT ENABLE_NOISY_WARNINGS)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-unused-function")
  endif()

  # ---------------------------------------------
  #      Define executable targets
  # ---------------------------------------------
  MACRO(OPENCV_DEFINE_GPU_EXAMPLE name srcs)
    set(the_target "example_${project}_${name}")
    add_executable(${the_target} ${srcs})

    target_link_libraries(${the_target} ${OPENCV_LINKER_LIBS} ${OPENCV_GPU_SAMPLES_REQUIRED_DEPS})

    if(HAVE_CUDA AND NOT ANDROID)
      target_link_libraries(${the_target} ${CUDA_CUDA_LIBRARY})
    endif()

    if(HAVE_opencv_nonfree)
      target_link_libraries(${the_target} opencv_nonfree)
    endif()

    if(HAVE_opencv_ocl)
      target_link_libraries(${the_target} opencv_ocl)
    endif()

    set_target_properties(${the_target} PROPERTIES
      OUTPUT_NAME "${project}-example-${name}"
      PROJECT_LABEL "(EXAMPLE_${project_upper}) ${name}")

    if(ENABLE_SOLUTION_FOLDERS)
      set_target_properties(${the_target} PROPERTIES FOLDER "samples//${project}")
    endif()

    if(WIN32)
      if(MSVC AND NOT BUILD_SHARED_LIBS)
        set_target_properties(${the_target} PROPERTIES LINK_FLAGS "/NODEFAULTLIB:atlthunk.lib /NODEFAULTLIB:atlsd.lib /DEBUG")
      endif()
      install(TARGETS ${the_target} RUNTIME DESTINATION "${OPENCV_SAMPLES_BIN_INSTALL_PATH}/${project}" COMPONENT samples)
    endif()
  ENDMACRO()

  # remove all matching elements from the list
  MACRO(list_filterout lst regex)
    foreach(item ${${lst}})
      if(item MATCHES "${regex}")
        list(REMOVE_ITEM ${lst} "${item}")
      endif()
    endforeach()
  ENDMACRO()

  file(GLOB all_samples RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} *.cpp)

  if(NOT HAVE_OPENGL)
    list(REMOVE_ITEM all_samples "opengl.cpp")
  endif()

  if(NOT HAVE_CUDA)
    list(REMOVE_ITEM all_samples "opticalflow_nvidia_api.cpp")
    list(REMOVE_ITEM all_samples "cascadeclassifier_nvidia_api.cpp")
    list(REMOVE_ITEM all_samples "driver_api_multi.cpp")
    list(REMOVE_ITEM all_samples "driver_api_stereo_multi.cpp")
  endif()
  if(NOT HAVE_CUDA
      OR NOT HAVE_TBB
      OR OpenCV_FOUND  # via find_package() there is no access to cvconfig.h
  )
    list(REMOVE_ITEM all_samples "pyrlk_optical_flow_multithreading.cpp")
  endif()

  foreach(sample_filename ${all_samples})
    get_filename_component(sample ${sample_filename} NAME_WE)
    file(GLOB sample_srcs RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} ${sample}.*)
    OPENCV_DEFINE_GPU_EXAMPLE(${sample} ${sample_srcs})
  endforeach()

  include("performance/CMakeLists.txt")
endif()

if (OCV_DEPENDENCIES_FOUND AND INSTALL_C_EXAMPLES AND NOT WIN32)
  file(GLOB install_list *.c *.cpp *.jpg *.png *.data makefile.* build_all.sh *.dsp *.cmd *.txt)
  if(NOT HAVE_OPENGL)
    list_filterout(install_list ".*opengl.cpp")
  endif()
  if(NOT HAVE_CUDA)
    list_filterout(install_list ".*opticalflow_nvidia_api.cpp")
    list_filterout(install_list ".*cascadeclassifier_nvidia_api.cpp")
    list_filterout(install_list ".*driver_api_multi.cpp")
    list_filterout(install_list ".*driver_api_stereo_multi.cpp")
  endif()
  if(NOT HAVE_CUDA OR NOT HAVE_TBB)
      list(REMOVE_ITEM install_list "pyrlk_optical_flow_multithreading.cpp")
  endif()
  install(FILES ${install_list}
          DESTINATION "${OPENCV_SAMPLES_SRC_INSTALL_PATH}/gpu"
          PERMISSIONS OWNER_READ OWNER_WRITE GROUP_READ WORLD_READ COMPONENT samples)
endif()
