file(GLOB TEST_OPS RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}" "test_*.py")
string(REPLACE ".py" "" TEST_OPS "${TEST_OPS}")
set(GC_ENVS FLAGS_eager_delete_tensor_gb=0.0 FLAGS_fast_eager_deletion_mode=1 FLAGS_memory_fraction_of_eager_deletion=1.0)
set(dist_ENVS http_proxy="" https_proxy="")

if(NOT WITH_DISTRIBUTE)
    list(REMOVE_ITEM TEST_OPS test_recv_op)
    list(REMOVE_ITEM TEST_OPS test_dist_transpiler)
    list(REMOVE_ITEM TEST_OPS test_simple_dist_transpiler)
    list(REMOVE_ITEM TEST_OPS test_listen_and_serv_op)
    LIST(REMOVE_ITEM TEST_OPS test_dist_mnist)
    LIST(REMOVE_ITEM TEST_OPS test_dist_mnist_fleetapi)
    LIST(REMOVE_ITEM TEST_OPS test_dist_mnist_dgc_nccl)
    LIST(REMOVE_ITEM TEST_OPS test_dist_mnist_hallreduce)
    LIST(REMOVE_ITEM TEST_OPS test_dist_mnist_multi_comm)
    LIST(REMOVE_ITEM TEST_OPS test_dist_mnist_ring_allreduce)
    LIST(REMOVE_ITEM TEST_OPS test_dist_mnist_backward_deps)
    LIST(REMOVE_ITEM TEST_OPS test_dist_mnist_lars)
    LIST(REMOVE_ITEM TEST_OPS test_dist_word2vec)
    LIST(REMOVE_ITEM TEST_OPS test_dist_ctr)
    LIST(REMOVE_ITEM TEST_OPS test_dist_simnet_bow)
    LIST(REMOVE_ITEM TEST_OPS test_dist_mnist_batch_merge)
    LIST(REMOVE_ITEM TEST_OPS test_dist_text_classification)
    LIST(REMOVE_ITEM TEST_OPS test_nce_remote_table_op)
    LIST(REMOVE_ITEM TEST_OPS test_hsigmoid_remote_table_op)
    LIST(REMOVE_ITEM TEST_OPS test_dist_fleet_ctr)
endif(NOT WITH_DISTRIBUTE)


if(NOT WITH_GPU OR WIN32)
    LIST(REMOVE_ITEM TEST_OPS test_c_comm_init_all_op)
    LIST(REMOVE_ITEM TEST_OPS test_allgather)
    LIST(REMOVE_ITEM TEST_OPS test_allreduce)
    LIST(REMOVE_ITEM TEST_OPS test_broadcast)
    LIST(REMOVE_ITEM TEST_OPS test_reducescatter)
endif()

if(WIN32)
    LIST(REMOVE_ITEM TEST_OPS test_boxps)
    LIST(REMOVE_ITEM TEST_OPS test_trainer_desc)
    LIST(REMOVE_ITEM TEST_OPS test_multiprocess_reader_exception)
    LIST(REMOVE_ITEM TEST_OPS test_avoid_twice_initialization)
endif()

LIST(REMOVE_ITEM TEST_OPS test_launch)

if (NOT ${WITH_GPU})
    LIST(REMOVE_ITEM TEST_OPS test_conv2d_fusion_op)
    LIST(REMOVE_ITEM TEST_OPS test_parallel_dygraph_mnist) # TODO(Yancey1989): parallel dygraph support CPU device in future
elseif(${CUDNN_VERSION} VERSION_LESS 7100)
    LIST(REMOVE_ITEM TEST_OPS test_conv2d_fusion_op)
endif()

if(NOT WITH_GPU OR WIN32)
    LIST(REMOVE_ITEM TEST_OPS test_pipeline)
endif()
list(REMOVE_ITEM TEST_OPS test_seq_concat_op) # FIXME(helin): https://github.com/PaddlePaddle/Paddle/issues/8290
list(REMOVE_ITEM TEST_OPS test_lstm_unit_op) # # FIXME(qijun) https://github.com/PaddlePaddle/Paddle/issues/5185
list(REMOVE_ITEM TEST_OPS test_cond_op) # FIXME(qijun): https://github.com/PaddlePaddle/Paddle/issues/5101#issuecomment-339814957


list(REMOVE_ITEM TEST_OPS op_test) # op_test is a helper python file, not a test
list(REMOVE_ITEM TEST_OPS decorator_helper) # decorator_helper is a helper python file, not a test
if(APPLE)
    if(NOT WITH_DISTRIBUTE)
        list(REMOVE_ITEM TEST_OPS test_desc_clone)
        list(REMOVE_ITEM TEST_OPS test_program_code)
    endif(NOT WITH_DISTRIBUTE)
    message(WARNING "These tests has been disabled in OSX before being fixed:\n test_fuse_elewise_add_act_pass \n test_detection_map_op \n test_dist_se_resnext_*")
    # this op is not support on mac
    list(REMOVE_ITEM TEST_OPS test_fusion_seqexpand_concat_fc_op)
    # TODO: add the unitest back when it fixed
    list(REMOVE_ITEM TEST_OPS test_detection_map_op)
    list(REMOVE_ITEM TEST_OPS test_dist_se_resnext_dgc)
    list(REMOVE_ITEM TEST_OPS test_dist_se_resnext_sync)
    list(REMOVE_ITEM TEST_OPS test_dist_se_resnext_async)
    list(REMOVE_ITEM TEST_OPS test_dist_se_resnext_sync_with_memopt)
    # TODO(tangwei12): add the unitest back when it fixed
    list(REMOVE_ITEM TEST_OPS test_dist_word2vec)
    list(REMOVE_ITEM TEST_OPS test_fuse_elewise_add_act_pass)
endif()
if(NOT WITH_MKLML)
    # this op is not support on openblas
    list(REMOVE_ITEM TEST_OPS test_fusion_seqexpand_concat_fc_op)
endif()

if(NOT WITH_MKL)
  list(REMOVE_ITEM TEST_OPS test_match_matrix_tensor_op)
  list(REMOVE_ITEM TEST_OPS test_var_conv_2d)
endif(NOT WITH_MKL)

if(WITH_GPU OR NOT WITH_MKLML)
    # matmul with multiple heads need MKL support
    LIST(REMOVE_ITEM TEST_OPS test_matmul_op_with_head)
endif()

function(py_test_modules TARGET_NAME)
  if(WITH_TESTING)
    set(options SERIAL)
    set(oneValueArgs "")
    set(multiValueArgs MODULES DEPS ENVS)
    cmake_parse_arguments(py_test_modules "${options}" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})

    if(WITH_COVERAGE)
      add_test(NAME ${TARGET_NAME}
          COMMAND ${CMAKE_COMMAND} -E env PYTHONPATH=${PADDLE_BINARY_DIR}/python ${py_test_modules_ENVS}
          COVERAGE_FILE=${PADDLE_BINARY_DIR}/python-coverage.data
          ${PYTHON_EXECUTABLE} -m coverage run --branch -p ${PADDLE_SOURCE_DIR}/tools/test_runner.py ${py_test_modules_MODULES}
          WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
    else()
      add_test(NAME ${TARGET_NAME}
          COMMAND ${CMAKE_COMMAND} -E env PYTHONPATH=${PADDLE_BINARY_DIR}/python ${py_test_modules_ENVS}
          ${PYTHON_EXECUTABLE} ${PADDLE_SOURCE_DIR}/tools/test_runner.py ${py_test_modules_MODULES}
          WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
    endif()

    if (py_test_modules_SERIAL)
        set_property(TEST ${TARGET_NAME} PROPERTY RUN_SERIAL 1)
    endif()

    set_tests_properties(${TARGET_NAME} PROPERTIES TIMEOUT 350)
  endif()
endfunction()

function(bash_test_modules TARGET_NAME)
    if(NOT WITH_TESTING)
        return()
    endif()

    set(options SERIAL)
    set(oneValueArgs "")
    set(multiValueArgs MODULES DEPS ENVS)
    cmake_parse_arguments(bash_test_modules "${options}" "${oneValueArgs}" "${multiValueArgs}" ${ARGN})

    message(STATUS "CMAKE_CURRENT_BINARY_DIR:" ${CMAKE_CURRENT_BINARY_DIR})

    add_test(NAME ${TARGET_NAME}
        COMMAND ${CMAKE_COMMAND} -E env PYTHONPATH=${PADDLE_BINARY_DIR}/python ${bash_test_modules_ENVS}
        bash ${CMAKE_CURRENT_BINARY_DIR}/${bash_test_modules_MODULES}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR})
    if (bash_test_modules_SERIAL)
        set_property(TEST ${TARGET_NAME} PROPERTY RUN_SERIAL 1)
    endif()
    set_tests_properties(${TARGET_NAME} PROPERTIES TIMEOUT 600)
endfunction()

list(REMOVE_ITEM TEST_OPS test_warpctc_op)

LIST(REMOVE_ITEM TEST_OPS test_lookup_remote_table_op)
LIST(REMOVE_ITEM TEST_OPS test_hsigmoid_remote_table_op)
LIST(REMOVE_ITEM TEST_OPS test_nce_remote_table_op)
LIST(REMOVE_ITEM TEST_OPS test_dist_train)
LIST(REMOVE_ITEM TEST_OPS test_listen_and_serv_op)
list(REMOVE_ITEM TEST_OPS test_dist_transpiler)
list(REMOVE_ITEM TEST_OPS test_parallel_executor_crf)
list(REMOVE_ITEM TEST_OPS test_parallel_executor_crf_auto_growth)
list(REMOVE_ITEM TEST_OPS test_parallel_executor_fetch_feed)
list(REMOVE_ITEM TEST_OPS test_dist_se_resnext_dgc)
list(REMOVE_ITEM TEST_OPS test_dist_se_resnext_sync)
list(REMOVE_ITEM TEST_OPS test_dist_se_resnext_async)
list(REMOVE_ITEM TEST_OPS test_dist_se_resnext_sync_with_memopt)
list(REMOVE_ITEM TEST_OPS test_dgc_op)
list(REMOVE_ITEM TEST_OPS test_dist_se_resnext_nccl)
list(REMOVE_ITEM TEST_OPS test_dist_transformer)
list(REMOVE_ITEM TEST_OPS test_parallel_executor_transformer)
list(REMOVE_ITEM TEST_OPS test_parallel_executor_transformer_auto_growth)
list(REMOVE_ITEM TEST_OPS test_bilinear_interp_op)
list(REMOVE_ITEM TEST_OPS test_nearest_interp_op)
list(REMOVE_ITEM TEST_OPS test_imperative_resnet)
list(REMOVE_ITEM TEST_OPS test_imperative_resnet_sorted_gradient)
list(REMOVE_ITEM TEST_OPS test_imperative_mnist_sorted_gradient)
list(REMOVE_ITEM TEST_OPS test_imperative_se_resnext)
list(REMOVE_ITEM TEST_OPS test_imperative_mnist)
list(REMOVE_ITEM TEST_OPS test_ir_memory_optimize_transformer)
list(REMOVE_ITEM TEST_OPS test_layers)
list(REMOVE_ITEM TEST_OPS test_parallel_executor_seresnext_base_cpu)
list(REMOVE_ITEM TEST_OPS test_parallel_executor_seresnext_with_reduce_cpu)
list(REMOVE_ITEM TEST_OPS test_parallel_executor_seresnext_with_fuse_all_reduce_cpu)
list(REMOVE_ITEM TEST_OPS test_imperative_ocr_attention_model)
list(REMOVE_ITEM TEST_OPS test_async_ssa_graph_executor_mnist)
list(REMOVE_ITEM TEST_OPS test_install_check)
list(REMOVE_ITEM TEST_OPS test_basic_gru_api)
list(REMOVE_ITEM TEST_OPS test_basic_gru_unit_op)
list(REMOVE_ITEM TEST_OPS test_basic_lstm_api)
list(REMOVE_ITEM TEST_OPS test_basic_lstm_unit_op)
list(REMOVE_ITEM TEST_OPS test_imperative_debug_string)

if (APPLE OR WIN32)
  list(REMOVE_ITEM TEST_OPS test_dataset)
  list(REMOVE_ITEM TEST_OPS test_dataset_dataloader)
endif()

# Some ops need to check results when gc is enabled
# Currently, only ops that register NoNeedBufferVarsInference need to do this test
set(TEST_OPS_WITH_GC
  test_affine_channel_op
  test_concat_op
  test_elementwise_add_op
  test_elementwise_sub_op
  test_fill_constant_batch_size_like_op
  test_fill_zeros_like2_op
  test_gather_op
  test_gather_nd_op
  test_gaussian_random_batch_size_like_op
  test_linear_chain_crf_op
  test_lod_reset_op
  test_lookup_table_op
  test_mean_op
  test_pad2d_op
  test_scatter_op
  test_sequence_concat
  test_seq_conv
  test_seq_pool
  test_sequence_expand_as
  test_sequence_expand
  test_sequence_pad_op
  test_sequence_unpad_op
  test_sequence_scatter_op
  test_sequence_slice_op
  test_slice_op
  test_space_to_depth_op
  test_squared_l2_distance_op
  test_uniform_random_batch_size_like_op)

foreach(TEST_OP ${TEST_OPS_WITH_GC})
  list(REMOVE_ITEM TEST_OPS ${TEST_OP})
  py_test_modules(${TEST_OP} MODULES ${TEST_OP} ENVS ${GC_ENVS})
endforeach()

foreach(TEST_OP ${TEST_OPS})
    py_test_modules(${TEST_OP} MODULES ${TEST_OP})
endforeach(TEST_OP)
py_test_modules(test_adam_op_multi_thread MODULES test_adam_op ENVS FLAGS_inner_op_parallelism=4)
py_test_modules(test_warpctc_op MODULES test_warpctc_op)
py_test_modules(test_bilinear_interp_op MODULES test_bilinear_interp_op ENVS ${GC_ENVS})
py_test_modules(test_nearest_interp_op MODULES test_nearest_interp_op ENVS ${GC_ENVS})
py_test_modules(test_imperative_resnet MODULES test_imperative_resnet ENVS
    FLAGS_cudnn_deterministic=1 SERIAL)
set_tests_properties(test_imperative_resnet PROPERTIES LABELS "RUN_TYPE=EXCLUSIVE" RUN_SERIAL TRUE)
py_test_modules(test_imperative_resnet_sorted_gradient MODULES test_imperative_resnet_sorted_gradient ENVS
        FLAGS_cudnn_deterministic=1 SERIAL)
set_tests_properties(test_imperative_resnet_sorted_gradient PROPERTIES LABELS "RUN_TYPE=EXCLUSIVE" RUN_SERIAL TRUE)
py_test_modules(test_imperative_mnist MODULES test_imperative_mnist ENVS
    FLAGS_cudnn_deterministic=1)
py_test_modules(test_imperative_mnist_sorted_gradient MODULES test_imperative_mnist_sorted_gradient ENVS
        FLAGS_cudnn_deterministic=1)
py_test_modules(test_imperative_se_resnext MODULES test_imperative_se_resnext ENVS
    FLAGS_cudnn_deterministic=1 SERIAL)
set_tests_properties(test_imperative_se_resnext PROPERTIES LABELS "RUN_TYPE=EXCLUSIVE" RUN_SERIAL TRUE)
py_test_modules(test_imperative_ocr_attention_model MODULES test_imperative_ocr_attention_model ENVS
        FLAGS_cudnn_deterministic=1 SERIAL)
py_test_modules(test_install_check MODULES test_install_check ENVS
        FLAGS_cudnn_deterministic=1 SERIAL)
set_tests_properties(test_install_check PROPERTIES LABELS "RUN_TYPE=DIST")
py_test_modules(test_imperative_debug_string MODULES test_imperative_debug_string ENVS FLAGS_dygraph_debug=1)
if(WITH_DISTRIBUTE)
    py_test_modules(test_dist_train MODULES test_dist_train ENVS ${dist_ENVS})
    py_test_modules(test_lookup_remote_table_op MODULES test_lookup_remote_table_op ENVS ${dist_ENVS})
    py_test_modules(test_hsigmoid_remote_table_op MODULES test_hsigmoid_remote_table_op ENVS ${dist_ENVS})
    py_test_modules(test_nce_remote_table_op MODULES test_nce_remote_table_op ENVS ${dist_ENVS})
    #py_test_modules(test_listen_and_serv_op MODULES test_listen_and_serv_op ENVS ${dist_ENVS})
    if(WITH_DGC)
        py_test_modules(test_dgc_op MODULES test_dgc_op)
    endif()
    if(NOT APPLE)
        bash_test_modules(test_listen_and_serv_op MODULES test_listen_and_serv.sh)
        set_tests_properties(test_listen_and_serv_op PROPERTIES TIMEOUT 100 LABELS "RUN_TYPE=EXCLUSIVE" RUN_SERIAL TRUE)
        set_tests_properties(test_nce_remote_table_op test_hsigmoid_remote_table_op test_dist_ctr test_dist_fleet_ctr test_dist_mnist_batch_merge  PROPERTIES LABELS "RUN_TYPE=EXCLUSIVE" RUN_SERIAL TRUE)

        set_tests_properties(test_dist_mnist PROPERTIES TIMEOUT 350 LABELS "RUN_TYPE=EXCLUSIVE" RUN_SERIAL TRUE)
        set_tests_properties(test_dist_mnist_dgc_nccl PROPERTIES TIMEOUT 350 LABELS "RUN_TYPE=EXCLUSIVE" RUN_SERIAL TRUE)
        set_tests_properties(test_dist_mnist_hallreduce PROPERTIES TIMEOUT 350 LABELS "RUN_TYPE=EXCLUSIVE" RUN_SERIAL TRUE)
        set_tests_properties(test_dist_mnist_multi_comm PROPERTIES TIMEOUT 350 LABELS "RUN_TYPE=EXCLUSIVE" RUN_SERIAL TRUE)
        set_tests_properties(test_dist_mnist_ring_allreduce PROPERTIES TIMEOUT 350 LABELS "RUN_TYPE=EXCLUSIVE" RUN_SERIAL TRUE)
        set_tests_properties(test_dist_mnist_backward_deps PROPERTIES TIMEOUT 350 LABELS "RUN_TYPE=EXCLUSIVE"  RUN_SERIAL TRUE)
        set_tests_properties(test_dist_mnist_fleetapi  PROPERTIES TIMEOUT 350 LABELS "RUN_TYPE=EXCLUSIVE" RUN_SERIAL TRUE)
        set_tests_properties(test_dist_mnist_lars PROPERTIES TIMEOUT 350 LABELS "RUN_TYPE=EXCLUSIVE" RUN_SERIAL TRUE)
        set_tests_properties(test_dist_word2vec PROPERTIES TIMEOUT 350 LABELS "RUN_TYPE=EXCLUSIVE" RUN_SERIAL TRUE )
        set_tests_properties(test_dist_simnet_bow PROPERTIES TIMEOUT 350 LABELS "RUN_TYPE=EXCLUSIVE" RUN_SERIAL TRUE)
        set_tests_properties(test_dist_text_classification PROPERTIES TIMEOUT 350 LABELS "RUN_TYPE=EXCLUSIVE"  RUN_SERIAL TRUE)

        list(REMOVE_ITEM TEST_OPS test_dist_se_resnext_dgc)
        list(REMOVE_ITEM TEST_OPS test_dist_se_resnext_sync)
	    list(REMOVE_ITEM TEST_OPS test_dist_se_resnext_async)
	    list(REMOVE_ITEM TEST_OPS test_dist_se_resnext_sync_with_memopt)

        py_test_modules(test_dist_se_resnext_dgc MODULES test_dist_se_resnext_dgc)
	    py_test_modules(test_dist_se_resnext_sync MODULES test_dist_se_resnext_sync)
        py_test_modules(test_dist_se_resnext_nccl MODULES test_dist_se_resnext_nccl)
        bash_test_modules(test_launch MODULES test_launch.sh)

        # FIXME(typhoonzero): add these tests back
        # py_test_modules(test_dist_transformer MODULES test_dist_transformer)
        # set_tests_properties(test_dist_transformer PROPERTIES TIMEOUT 1000)
        set_tests_properties(test_dist_se_resnext_dgc PROPERTIES LABELS "RUN_TYPE=EXCLUSIVE" RUN_SERIAL TRUE)
        set_tests_properties(test_dist_se_resnext_sync PROPERTIES LABELS "RUN_TYPE=EXCLUSIVE" RUN_SERIAL TRUE)
        set_tests_properties(test_dist_se_resnext_nccl PROPERTIES LABELS "RUN_TYPE=EXCLUSIVE" RUN_SERIAL TRUE)
    endif(NOT APPLE)
    # py_test_modules(test_dist_transpiler MODULES test_dist_transpiler)
endif()

py_test_modules(test_parallel_executor_crf MODULES test_parallel_executor_crf)
py_test_modules(test_parallel_executor_crf_auto_growth MODULES test_parallel_executor_crf_auto_growth ENVS FLAGS_allocator_strategy=auto_growth)
py_test_modules(test_parallel_executor_fetch_feed MODULES test_parallel_executor_fetch_feed)
set_tests_properties(test_parallel_executor_fetch_feed PROPERTIES TIMEOUT 450)
py_test_modules(test_parallel_executor_transformer MODULES test_parallel_executor_transformer)
py_test_modules(test_parallel_executor_transformer_auto_growth MODULES test_parallel_executor_transformer_auto_growth ENVS FLAGS_allocator_strategy=auto_growth)
py_test_modules(test_layers MODULES test_layers ENVS FLAGS_cudnn_deterministic=1)
py_test_modules(test_parallel_executor_seresnext_base_cpu MODULES test_parallel_executor_seresnext_base_cpu)
py_test_modules(test_parallel_executor_seresnext_with_reduce_cpu MODULES test_parallel_executor_seresnext_with_reduce_cpu)
py_test_modules(test_parallel_executor_seresnext_with_fuse_all_reduce_cpu MODULES test_parallel_executor_seresnext_with_fuse_all_reduce_cpu)

if(NOT WIN32)
    py_test_modules(test_ir_memory_optimize_transformer MODULES test_ir_memory_optimize_transformer)
endif()

set_tests_properties(test_parallel_executor_seresnext_base_cpu PROPERTIES TIMEOUT 900)
set_tests_properties(test_parallel_executor_seresnext_with_reduce_cpu PROPERTIES TIMEOUT 750)
set_tests_properties(test_parallel_executor_seresnext_with_fuse_all_reduce_cpu PROPERTIES TIMEOUT 750)

if (WITH_NGRAPH)
    add_subdirectory(ngraph)
endif()

if (WITH_MKLDNN)
    add_subdirectory(mkldnn)
endif()

if (WITH_TESTING)
    set_property(TEST test_parallel_executor_mnist PROPERTY ENVIRONMENT GLOG_vmodule=scope_buffered_ssa_graph_executor=5)
endif()

set_tests_properties(test_parallel_executor_test_while_train test_parallel_executor_mnist
        test_parallel_executor_seresnext_base_gpu test_parallel_executor_seresnext_with_reduce_gpu
        test_parallel_executor_seresnext_with_fuse_all_reduce_gpu
        test_parallel_executor_crf test_sync_batch_norm_op
        test_parallel_executor_feed_persistable_var
        test_parallel_executor_crf_auto_growth test_buffer_shared_memory_reuse_pass_and_fuse_optimization_op_pass
        test_buffer_shared_memory_reuse_pass PROPERTIES LABELS "RUN_TYPE=DIST")
