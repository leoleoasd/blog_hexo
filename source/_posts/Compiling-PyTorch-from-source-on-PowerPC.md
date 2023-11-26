---
title: Compiling PyTorch from source on a PowerPC server
typora-root-url: ./..
typora-copy-images-to: ../medias/${filename}
date: 2023-11-26 15:30:28
tags:
---

PowerPC is IBM's architecture and nobody but IBM is using it anymore. IBM have their own build of `PyTorch` and `TensorFlow` to powerpc, which called the `powerai` bundle, however the last update was 4 years ago and it only supports `pytorch 1.3`.

So I HAVE to build PyTorch myself. Building softwares is fairly easy on Linux, as long as you know to **put the right thing in the right place**.

I choose `1.12.1` because it seems that it's the last version that supports cuda 11.2. With some modification to the code, `pytorch 2.0.1` seems to compile but eventually fail with an error that I don't have time to figure out how to solve.

These steps is used inside a cluster that I don't have `sudo` access. Running cuda installer will result in `segmentation fault` but luckily the system admin have cuda 11.2 installed at `/usr/local/cuda`. The system don't have `CUDNN` though, however it is possible to use `CUDNN` from conda-forge.

1. install miniforge

2. create a new mamba environment, with `mamba create -n <your_env_name_here> python=3.10 cudatoolkit=11.2 cudnn libopenblas blas cmake ninja`

3. clone PyTorch; git checkout v1.12.1; IGNORE pytorch document that asks you to install mkl.

4. ```
   cd pytorch
   git submodule sync
   git submodule update --init --recursive

5. ```
   export CUDA_HOME=/usr/local/cuda
   export LD_LIBRARY_PATH=$CUDA_HOME/lib64:$CONDA_PREFIX/lib:$LD_LIBRARY_PATH
   ```

6. ```
   pip install -r requirements.txt
   ```

7. ```
   export MAX_JOBS=40 # limits the total number of threads. 
   									 # Each PPC core have 4 hardware threads but will cause too much stress to the file system, so I'm running one thread per core.
   python setup.py build
   ```

8. Make sure `USE_CUDA` is `on`:

   ```
   -- ******** Summary ********
   -- General:
   --   CMake version         : 3.27.8
   --   CMake command         : $CONDA_PREFIX/bin/cmake
   --   System                : Linux
   --   C++ compiler          : /usr/bin/c++
   --   C++ compiler id       : GNU
   --   C++ compiler version  : 8.4.1
   --   Using ccache if found : ON
   --   Found ccache          : CCACHE_PROGRAM-NOTFOUND
   --   CXX flags             :  -Wno-deprecated -fvisibility-inlines-hidden -fopenmp -DNDEBUG -DUSE_KINETO -DSYMBOLICATE_MOBILE_DEBUG_HANDLE -DEDGE_PROFILER_USE_KINETO -O2 -fPIC -Wno-narrowing -Wall -Wextra -Werror=return-type -Wno-missing-field-initializers -Wno-type-limits -Wno-array-bounds -Wno-unknown-pragmas -Wno-unused-parameter -Wno-unused-function -Wno-unused-result -Wno-unused-local-typedefs -Wno-strict-overflow -Wno-strict-aliasing -Wno-error=deprecated-declarations -Wno-stringop-overflow -Wno-psabi -Wno-error=pedantic -Wno-error=redundant-decls -Wno-error=old-style-cast -fdiagnostics-color=always -faligned-new -Wno-unused-but-set-variable -Wno-maybe-uninitialized -fno-math-errno -fno-trapping-math -Werror=format -Werror=cast-function-type -Wno-stringop-overflow
   --   Build type            : Release
   --   Compile definitions   : ONNX_ML=1;ONNXIFI_ENABLE_EXT=1;ONNX_NAMESPACE=onnx_torch;HAVE_MMAP=1;_FILE_OFFSET_BITS=64;HAVE_SHM_OPEN=1;HAVE_SHM_UNLINK=1;HAVE_MALLOC_USABLE_SIZE=1;USE_EXTERNAL_MZCRC;MINIZ_DISABLE_ZIP_READER_CRC32_CHECKS
   --   CMAKE_PREFIX_PATH     : $CONDA_PREFIX/lib/python3.10/site-packages;/usr/local/cuda
   --   CMAKE_INSTALL_PREFIX  : pytorch/torch
   --   USE_GOLD_LINKER       : OFF
   --
   --   TORCH_VERSION         : 1.12.0
   --   CAFFE2_VERSION        : 1.12.0
   --   BUILD_CAFFE2          : OFF
   --   BUILD_CAFFE2_OPS      : OFF
   --   BUILD_CAFFE2_MOBILE   : OFF
   --   BUILD_STATIC_RUNTIME_BENCHMARK: OFF
   --   BUILD_TENSOREXPR_BENCHMARK: OFF
   --   BUILD_NVFUSER_BENCHMARK: ON
   --   BUILD_BINARY          : OFF
   --   BUILD_CUSTOM_PROTOBUF : ON
   --     Link local protobuf : ON
   --   BUILD_DOCS            : OFF
   --   BUILD_PYTHON          : True
   --     Python version      : 3.10.13
   --     Python executable   : $CONDA_PREFIX/bin/python
   --     Pythonlibs version  : 3.10.13
   --     Python library      : $CONDA_PREFIX/lib/libpython3.10.a
   --     Python includes     : $CONDA_PREFIX/include/python3.10
   --     Python site-packages: lib/python3.10/site-packages
   --   BUILD_SHARED_LIBS     : ON
   --   CAFFE2_USE_MSVC_STATIC_RUNTIME     : OFF
   --   BUILD_TEST            : True
   --   BUILD_JNI             : OFF
   --   BUILD_MOBILE_AUTOGRAD : OFF
   --   BUILD_LITE_INTERPRETER: OFF
   --   INTERN_BUILD_MOBILE   :
   --   USE_BLAS              : 1
   --     BLAS                : open
   --     BLAS_HAS_SBGEMM     : 1
   --   USE_LAPACK            : 1
   --     LAPACK              : open
   --   USE_ASAN              : OFF
   --   USE_CPP_CODE_COVERAGE : OFF
   --   USE_CUDA              : ON
   --     Split CUDA          : OFF
   --     CUDA static link    : OFF
   --     USE_CUDNN           : ON
   --     USE_EXPERIMENTAL_CUDNN_V8_API: ON
   --     CUDA version        : 11.2
   --     cuDNN version       : 8.8.0
   --     CUDA root directory : /usr/local/cuda
   --     CUDA library        : /usr/local/cuda/lib64/stubs/libcuda.so
   --     cudart library      : /usr/local/cuda/lib64/libcudart.so
   --     cublas library      : /usr/local/cuda/lib64/libcublas.so
   --     cufft library       : /usr/local/cuda/lib64/libcufft.so
   --     curand library      : /usr/local/cuda/lib64/libcurand.so
   --     cuDNN library       : $CONDA_PREFIX/lib/libcudnn.so
   --     nvrtc               : /usr/local/cuda/lib64/libnvrtc.so
   --     CUDA include path   : /usr/local/cuda/include
   --     NVCC executable     : /usr/local/cuda/bin/nvcc
   --     CUDA compiler       : /usr/local/cuda/bin/nvcc
   --     CUDA flags          :  -Xfatbin -compress-all -DONNX_NAMESPACE=onnx_torch -gencode arch=compute_70,code=sm_70 -Xcudafe --diag_suppress=cc_clobber_ignored,--diag_suppress=integer_sign_change,--diag_suppress=useless_using_declaration,--diag_suppress=set_but_not_used,--diag_suppress=field_without_dll_interface,--diag_suppress=base_class_has_different_dll_interface,--diag_suppress=dll_interface_conflict_none_assumed,--diag_suppress=dll_interface_conflict_dllexport_assumed,--diag_suppress=implicit_return_from_non_void_function,--diag_suppress=unsigned_compare_with_zero,--diag_suppress=declared_but_not_referenced,--diag_suppress=bad_friend_decl --expt-relaxed-constexpr --expt-extended-lambda  -Wno-deprecated-gpu-targets --expt-extended-lambda -DCUDA_HAS_FP16=1 -D__CUDA_NO_HALF_OPERATORS__ -D__CUDA_NO_HALF_CONVERSIONS__ -D__CUDA_NO_HALF2_OPERATORS__ -D__CUDA_NO_BFLOAT16_CONVERSIONS__
   --     CUDA host compiler  :
   --     CUDA --device-c     : OFF
   --     USE_TENSORRT        : OFF
   --   USE_ROCM              : OFF
   --   USE_EIGEN_FOR_BLAS    : ON
   --   USE_FBGEMM            : OFF
   --     USE_FAKELOWP          : OFF
   --   USE_KINETO            : ON
   --   USE_FFMPEG            : OFF
   --   USE_GFLAGS            : OFF
   --   USE_GLOG              : OFF
   --   USE_LEVELDB           : OFF
   --   USE_LITE_PROTO        : OFF
   --   USE_LMDB              : OFF
   --   USE_METAL             : OFF
   --   USE_PYTORCH_METAL     : OFF
   --   USE_PYTORCH_METAL_EXPORT     : OFF
   --   USE_MPS               : OFF
   --   USE_FFTW              : OFF
   --   USE_MKL               : OFF
   --   USE_MKLDNN            : OFF
   --   USE_NCCL              : ON
   --     USE_SYSTEM_NCCL     : OFF
   --     USE_NCCL_WITH_UCC   : OFF
   --   USE_NNPACK            : OFF
   --   USE_NUMPY             : ON
   --   USE_OBSERVERS         : ON
   --   USE_OPENCL            : OFF
   --   USE_OPENCV            : OFF
   --   USE_OPENMP            : ON
   --   USE_TBB               : OFF
   --   USE_VULKAN            : OFF
   --   USE_PROF              : OFF
   --   USE_QNNPACK           : OFF
   --   USE_PYTORCH_QNNPACK   : OFF
   --   USE_XNNPACK           : OFF
   --   USE_REDIS             : OFF
   --   USE_ROCKSDB           : OFF
   --   USE_ZMQ               : OFF
   --   USE_DISTRIBUTED       : ON
   --     USE_MPI               : OFF
   --     USE_GLOO              : ON
   --     USE_GLOO_WITH_OPENSSL : OFF
   --     USE_TENSORPIPE        : ON
   --   USE_DEPLOY           : OFF
   --   Public Dependencies  : caffe2::Threads
   --   Private Dependencies : cpuinfo;fp16;tensorpipe;gloo;foxi_loader;rt;fmt::fmt-header-only;kineto;gcc_s;gcc;dl
   --   USE_COREML_DELEGATE     : OFF
   --   BUILD_LAZY_TS_BACKEND   : ON
   -- Configuring done (34.5s)
   ```

9. ```
   python3 setup.py bdist_wheel
   pip install dist/torch*.whl
   cd ..
   python3 -c 'import torch'
   ```

10. Verify installation

    ```
    $ python3
    Python 3.10.13 | packaged by conda-forge | (main, Oct 26 2023, 18:10:55) [GCC 12.3.0] on linux
    Type "help", "copyright", "credits" or "license" for more information.
    >>> import torch
    >>> torch.__version__
    '1.12.0a0+git664058f'
    >>> torch.cuda.is_available()
    True
    >>> a = torch.tensor([1,2,3]).cuda()
    >>> for i in range(torch.cuda.device_count()):
    ...     a = a.to(f'cuda:{i}')
    ...     b = torch.tensor([2,3,4], device=a.device)
    ...     a*b
    ...
    tensor([ 2,  6, 12], device='cuda:0')
    tensor([ 2,  6, 12], device='cuda:1')
    tensor([ 2,  6, 12], device='cuda:2')
    tensor([ 2,  6, 12], device='cuda:3')
    tensor([ 2,  6, 12], device='cuda:4')
    tensor([ 2,  6, 12], device='cuda:5')
    >>>
    ```

11. Enjoy an empty cluster that nobody but you know how to use!
