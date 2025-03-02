#!/usr/bin/groovy
// -*- mode: groovy -*-
// Jenkins pipeline
// See documents at https://jenkins.io/doc/book/pipeline/jenkinsfile/

// Command to run command inside a docker container
dockerRun = 'tests/ci_build/ci_build.sh'

// Which CUDA version to use when building reference distribution wheel
ref_cuda_ver = '11.0'

import groovy.transform.Field

@Field
def commit_id   // necessary to pass a variable from one stage to another

pipeline {
  // Each stage specify its own agent
  agent none

  environment {
    DOCKER_CACHE_ECR_ID = '492475357299'
    DOCKER_CACHE_ECR_REGION = 'us-west-2'
  }

  // Setup common job properties
  options {
    ansiColor('xterm')
    timestamps()
    timeout(time: 240, unit: 'MINUTES')
    buildDiscarder(logRotator(numToKeepStr: '10'))
    preserveStashes()
  }

  // Build stages
  stages {
    stage('Jenkins Linux: Initialize') {
      agent { label 'job_initializer' }
      steps {
        script {
          def buildNumber = env.BUILD_NUMBER as int
          if (buildNumber > 1) milestone(buildNumber - 1)
          milestone(buildNumber)

          checkoutSrcs()
          commit_id = "${GIT_COMMIT}"
        }
        sh 'python3 tests/jenkins_get_approval.py'
        stash name: 'srcs'
      }
    }
    stage('Jenkins Linux: Build') {
      agent none
      steps {
        script {
          parallel ([
            'clang-tidy': { ClangTidy() },
            'build-cpu': { BuildCPU() },
            'build-cpu-arm64': { BuildCPUARM64() },
            'build-cpu-rabit-mock': { BuildCPUMock() },
            // Build reference, distribution-ready Python wheel with CUDA 11.0
            // using CentOS 7 image
            'build-gpu-cuda11.0': { BuildCUDA(cuda_version: '11.0', build_rmm: true) },
            'build-gpu-rpkg': { BuildRPackageWithCUDA(cuda_version: '11.0') },
            'build-jvm-packages-gpu-cuda11.0': { BuildJVMPackagesWithCUDA(spark_version: '3.0.1', cuda_version: '11.0') },
            'build-jvm-packages': { BuildJVMPackages(spark_version: '3.0.1') },
            'build-jvm-doc': { BuildJVMDoc() }
          ])
        }
      }
    }
    stage('Jenkins Linux: Test') {
      agent none
      steps {
        script {
          parallel ([
            'test-python-cpu': { TestPythonCPU() },
            'test-python-cpu-arm64': { TestPythonCPUARM64() },
            // artifact_cuda_version doesn't apply to RMM tests; RMM tests will always match CUDA version between artifact and host env
            'test-python-gpu-cuda11.0': { TestPythonGPU(artifact_cuda_version: '11.0', host_cuda_version: '11.0', test_rmm: true) },
            'test-python-mgpu-cuda11.0': { TestPythonGPU(artifact_cuda_version: '11.0', host_cuda_version: '11.0', multi_gpu: true, test_rmm: true) },
            'test-cpp-gpu-cuda11.0': { TestCppGPU(artifact_cuda_version: '11.0', host_cuda_version: '11.0', test_rmm: true) },
            'test-jvm-jdk8': { CrossTestJVMwithJDK(jdk_version: '8', spark_version: '3.0.0') }
          ])
        }
      }
    }
    stage('Jenkins Linux: Deploy') {
      agent none
      steps {
        script {
          parallel ([
            'deploy-jvm-packages': { DeployJVMPackages(spark_version: '3.0.0') }
          ])
        }
      }
    }
  }
}

// check out source code from git
def checkoutSrcs() {
  retry(5) {
    try {
      timeout(time: 2, unit: 'MINUTES') {
        checkout scm
        sh 'git submodule update --init'
      }
    } catch (exc) {
      deleteDir()
      error "Failed to fetch source codes"
    }
  }
}

def GetCUDABuildContainerType(cuda_version) {
  return (cuda_version == ref_cuda_ver) ? 'gpu_build_centos7' : 'gpu_build'
}

def ClangTidy() {
  node('linux && cpu_build') {
    unstash name: 'srcs'
    echo "Running clang-tidy job..."
    def container_type = "clang_tidy"
    def docker_binary = "docker"
    def dockerArgs = "--build-arg CUDA_VERSION_ARG=10.1"
    sh """
    ${dockerRun} ${container_type} ${docker_binary} ${dockerArgs} python3 tests/ci_build/tidy.py
    """
    deleteDir()
  }
}

def BuildCPU() {
  node('linux && cpu') {
    unstash name: 'srcs'
    echo "Build CPU"
    def container_type = "cpu"
    def docker_binary = "docker"
    sh """
    ${dockerRun} ${container_type} ${docker_binary} rm -fv dmlc-core/include/dmlc/build_config_default.h
      # This step is not necessary, but here we include it, to ensure that DMLC_CORE_USE_CMAKE flag is correctly propagated
      # We want to make sure that we use the configured header build/dmlc/build_config.h instead of include/dmlc/build_config_default.h.
      # See discussion at https://github.com/dmlc/xgboost/issues/5510
    ${dockerRun} ${container_type} ${docker_binary} tests/ci_build/build_via_cmake.sh -DPLUGIN_DENSE_PARSER=ON
    ${dockerRun} ${container_type} ${docker_binary} bash -c "cd build && ctest --extra-verbose"
    """
    // Sanitizer test
    def docker_extra_params = "CI_DOCKER_EXTRA_PARAMS_INIT='-e ASAN_SYMBOLIZER_PATH=/usr/bin/llvm-symbolizer -e ASAN_OPTIONS=symbolize=1 -e UBSAN_OPTIONS=print_stacktrace=1:log_path=ubsan_error.log --cap-add SYS_PTRACE'"
    sh """
    ${dockerRun} ${container_type} ${docker_binary} tests/ci_build/build_via_cmake.sh -DUSE_SANITIZER=ON -DENABLED_SANITIZERS="address;leak;undefined" \
      -DCMAKE_BUILD_TYPE=Debug -DSANITIZER_PATH=/usr/lib/x86_64-linux-gnu/
    ${docker_extra_params} ${dockerRun} ${container_type} ${docker_binary} bash -c "cd build && ctest --exclude-regex AllTestsInDMLCUnitTests --extra-verbose"
    """

    stash name: 'xgboost_cli', includes: 'xgboost'
    deleteDir()
  }
}

def BuildCPUARM64() {
  node('linux && arm64') {
    unstash name: 'srcs'
    echo "Build CPU ARM64"
    def container_type = "aarch64"
    def docker_binary = "docker"
    def wheel_tag = "manylinux2014_aarch64"
    sh """
    ${dockerRun} ${container_type} ${docker_binary} tests/ci_build/build_via_cmake.sh --conda-env=aarch64_test -DOPEN_MP:BOOL=ON -DHIDE_CXX_SYMBOL=ON
    ${dockerRun} ${container_type} ${docker_binary} bash -c "cd build && ctest --extra-verbose"
    ${dockerRun} ${container_type} ${docker_binary} bash -c "cd python-package && rm -rf dist/* && python setup.py bdist_wheel --universal"
    ${dockerRun} ${container_type} ${docker_binary} python tests/ci_build/rename_whl.py python-package/dist/*.whl ${commit_id} ${wheel_tag}
    ${dockerRun} ${container_type} ${docker_binary} bash -c "auditwheel repair --plat ${wheel_tag} python-package/dist/*.whl && python tests/ci_build/rename_whl.py wheelhouse/*.whl ${commit_id} ${wheel_tag}"
    mv -v wheelhouse/*.whl python-package/dist/
    # Make sure that libgomp.so is vendored in the wheel
    ${dockerRun} ${container_type} ${docker_binary} bash -c "unzip -l python-package/dist/*.whl | grep libgomp || exit -1"
    """
    echo 'Stashing Python wheel...'
    stash name: "xgboost_whl_arm64_cpu", includes: 'python-package/dist/*.whl'
    if (env.BRANCH_NAME == 'master' || env.BRANCH_NAME.startsWith('release')) {
      echo 'Uploading Python wheel...'
      path = ("${BRANCH_NAME}" == 'master') ? '' : "${BRANCH_NAME}/"
      s3Upload bucket: 'xgboost-nightly-builds', path: path, acl: 'PublicRead', workingDir: 'python-package/dist', includePathPattern:'**/*.whl'
    }
    stash name: 'xgboost_cli_arm64', includes: 'xgboost'
    deleteDir()
  }
}

def BuildCPUMock() {
  node('linux && cpu') {
    unstash name: 'srcs'
    echo "Build CPU with rabit mock"
    def container_type = "cpu"
    def docker_binary = "docker"
    sh """
    ${dockerRun} ${container_type} ${docker_binary} tests/ci_build/build_mock_cmake.sh
    """
    echo 'Stashing rabit C++ test executable (xgboost)...'
    stash name: 'xgboost_rabit_tests', includes: 'xgboost'
    deleteDir()
  }
}

def BuildCUDA(args) {
  node('linux && cpu_build') {
    unstash name: 'srcs'
    echo "Build with CUDA ${args.cuda_version}"
    def container_type = GetCUDABuildContainerType(args.cuda_version)
    def docker_binary = "docker"
    def docker_args = "--build-arg CUDA_VERSION_ARG=${args.cuda_version}"
    def arch_flag = ""
    if (env.BRANCH_NAME != 'master' && !(env.BRANCH_NAME.startsWith('release'))) {
      arch_flag = "-DGPU_COMPUTE_VER=75"
    }
    def wheel_tag = "manylinux2014_x86_64"
    sh """
    ${dockerRun} ${container_type} ${docker_binary} ${docker_args} tests/ci_build/build_via_cmake.sh -DUSE_CUDA=ON -DUSE_NCCL=ON -DOPEN_MP:BOOL=ON -DHIDE_CXX_SYMBOLS=ON ${arch_flag}
    ${dockerRun} ${container_type} ${docker_binary} ${docker_args} bash -c "cd python-package && rm -rf dist/* && python setup.py bdist_wheel --universal"
    ${dockerRun} ${container_type} ${docker_binary} ${docker_args} python tests/ci_build/rename_whl.py python-package/dist/*.whl ${commit_id} ${wheel_tag}
    """
    if (args.cuda_version == ref_cuda_ver) {
      sh """
      ${dockerRun} auditwheel_x86_64 ${docker_binary} auditwheel repair --plat ${wheel_tag} python-package/dist/*.whl
      ${dockerRun} ${container_type} ${docker_binary} ${docker_args} python tests/ci_build/rename_whl.py wheelhouse/*.whl ${commit_id} ${wheel_tag}
      mv -v wheelhouse/*.whl python-package/dist/
      # Make sure that libgomp.so is vendored in the wheel
      ${dockerRun} auditwheel_x86_64 ${docker_binary} bash -c "unzip -l python-package/dist/*.whl | grep libgomp  || exit -1"
      """
    }
    echo 'Stashing Python wheel...'
    stash name: "xgboost_whl_cuda${args.cuda_version}", includes: 'python-package/dist/*.whl'
    if (args.cuda_version == ref_cuda_ver && (env.BRANCH_NAME == 'master' || env.BRANCH_NAME.startsWith('release'))) {
      echo 'Uploading Python wheel...'
      path = ("${BRANCH_NAME}" == 'master') ? '' : "${BRANCH_NAME}/"
      s3Upload bucket: 'xgboost-nightly-builds', path: path, acl: 'PublicRead', workingDir: 'python-package/dist', includePathPattern:'**/*.whl'
    }
    echo 'Stashing C++ test executable (testxgboost)...'
    stash name: "xgboost_cpp_tests_cuda${args.cuda_version}", includes: 'build/testxgboost'
    if (args.build_rmm) {
      echo "Build with CUDA ${args.cuda_version} and RMM"
      container_type = "rmm"
      docker_binary = "docker"
      docker_args = "--build-arg CUDA_VERSION_ARG=${args.cuda_version}"
      sh """
      rm -rf build/
      ${dockerRun} ${container_type} ${docker_binary} ${docker_args} tests/ci_build/build_via_cmake.sh --conda-env=gpu_test -DUSE_CUDA=ON -DUSE_NCCL=ON -DPLUGIN_RMM=ON -DBUILD_WITH_CUDA_CUB=ON ${arch_flag}
      ${dockerRun} ${container_type} ${docker_binary} ${docker_args} bash -c "cd python-package && rm -rf dist/* && python setup.py bdist_wheel --universal"
      ${dockerRun} ${container_type} ${docker_binary} ${docker_args} python tests/ci_build/rename_whl.py python-package/dist/*.whl ${commit_id} manylinux2014_x86_64
      """
      echo 'Stashing Python wheel...'
      stash name: "xgboost_whl_rmm_cuda${args.cuda_version}", includes: 'python-package/dist/*.whl'
      echo 'Stashing C++ test executable (testxgboost)...'
      stash name: "xgboost_cpp_tests_rmm_cuda${args.cuda_version}", includes: 'build/testxgboost'
    }
    deleteDir()
  }
}

def BuildRPackageWithCUDA(args) {
  node('linux && cpu_build') {
    unstash name: 'srcs'
    def container_type = 'gpu_build_r_centos7'
    def docker_binary = "docker"
    def docker_args = "--build-arg CUDA_VERSION_ARG=${args.cuda_version}"
    if (env.BRANCH_NAME == 'master' || env.BRANCH_NAME.startsWith('release')) {
      sh """
      ${dockerRun} ${container_type} ${docker_binary} ${docker_args} tests/ci_build/build_r_pkg_with_cuda.sh ${commit_id}
      """
      echo 'Uploading R tarball...'
      path = ("${BRANCH_NAME}" == 'master') ? '' : "${BRANCH_NAME}/"
      s3Upload bucket: 'xgboost-nightly-builds', path: path, acl: 'PublicRead', includePathPattern:'xgboost_r_gpu_linux_*.tar.gz'
    }
    deleteDir()
  }
}

def BuildJVMPackagesWithCUDA(args) {
  node('linux && mgpu') {
    unstash name: 'srcs'
    echo "Build XGBoost4J-Spark with Spark ${args.spark_version}, CUDA ${args.cuda_version}"
    def container_type = "jvm_gpu_build"
    def docker_binary = "nvidia-docker"
    def docker_args = "--build-arg CUDA_VERSION_ARG=${args.cuda_version}"
    def arch_flag = ""
    if (env.BRANCH_NAME != 'master' && !(env.BRANCH_NAME.startsWith('release'))) {
      arch_flag = "-DGPU_COMPUTE_VER=75"
    }
    // Use only 4 CPU cores
    def docker_extra_params = "CI_DOCKER_EXTRA_PARAMS_INIT='--cpuset-cpus 0-3'"
    sh """
    ${docker_extra_params} ${dockerRun} ${container_type} ${docker_binary} ${docker_args} tests/ci_build/build_jvm_packages.sh ${args.spark_version} -Duse.cuda=ON $arch_flag
    """
    echo "Stashing XGBoost4J JAR with CUDA ${args.cuda_version} ..."
    stash name: 'xgboost4j_jar_gpu', includes: "jvm-packages/xgboost4j-gpu/target/*.jar,jvm-packages/xgboost4j-spark-gpu/target/*.jar"
    deleteDir()
  }
}

def BuildJVMPackages(args) {
  node('linux && cpu') {
    unstash name: 'srcs'
    echo "Build XGBoost4J-Spark with Spark ${args.spark_version}"
    def container_type = "jvm"
    def docker_binary = "docker"
    // Use only 4 CPU cores
    def docker_extra_params = "CI_DOCKER_EXTRA_PARAMS_INIT='--cpuset-cpus 0-3'"
    sh """
    ${docker_extra_params} ${dockerRun} ${container_type} ${docker_binary} tests/ci_build/build_jvm_packages.sh ${args.spark_version}
    """
    echo 'Stashing XGBoost4J JAR...'
    stash name: 'xgboost4j_jar', includes: "jvm-packages/xgboost4j/target/*.jar,jvm-packages/xgboost4j-spark/target/*.jar,jvm-packages/xgboost4j-example/target/*.jar"
    deleteDir()
  }
}

def BuildJVMDoc() {
  node('linux && cpu') {
    unstash name: 'srcs'
    echo "Building JVM doc..."
    def container_type = "jvm"
    def docker_binary = "docker"
    sh """
    ${dockerRun} ${container_type} ${docker_binary} tests/ci_build/build_jvm_doc.sh ${BRANCH_NAME}
    """
    if (env.BRANCH_NAME == 'master' || env.BRANCH_NAME.startsWith('release')) {
      echo 'Uploading doc...'
      s3Upload file: "jvm-packages/${BRANCH_NAME}.tar.bz2", bucket: 'xgboost-docs', acl: 'PublicRead', path: "${BRANCH_NAME}.tar.bz2"
    }
    deleteDir()
  }
}

def TestPythonCPU() {
  node('linux && cpu') {
    unstash name: "xgboost_whl_cuda${ref_cuda_ver}"
    unstash name: 'srcs'
    unstash name: 'xgboost_cli'
    echo "Test Python CPU"
    def container_type = "cpu"
    def docker_binary = "docker"
    sh """
    ${dockerRun} ${container_type} ${docker_binary} tests/ci_build/test_python.sh cpu
    """
    deleteDir()
  }
}

def TestPythonCPUARM64() {
  node('linux && arm64') {
    unstash name: "xgboost_whl_arm64_cpu"
    unstash name: 'srcs'
    unstash name: 'xgboost_cli_arm64'
    echo "Test Python CPU ARM64"
    def container_type = "aarch64"
    def docker_binary = "docker"
    sh """
    ${dockerRun} ${container_type} ${docker_binary} tests/ci_build/test_python.sh cpu-arm64
    """
    deleteDir()
  }
}

def TestPythonGPU(args) {
  def nodeReq = (args.multi_gpu) ? 'linux && mgpu' : 'linux && gpu'
  def artifact_cuda_version = (args.artifact_cuda_version) ?: ref_cuda_ver
  node(nodeReq) {
    unstash name: "xgboost_whl_cuda${artifact_cuda_version}"
    unstash name: "xgboost_cpp_tests_cuda${artifact_cuda_version}"
    unstash name: 'srcs'
    echo "Test Python GPU: CUDA ${args.host_cuda_version}"
    def container_type = "gpu"
    def docker_binary = "nvidia-docker"
    def docker_args = "--build-arg CUDA_VERSION_ARG=${args.host_cuda_version}"
    def mgpu_indicator = (args.multi_gpu) ? 'mgpu' : 'gpu'
    // Allocate extra space in /dev/shm to enable NCCL
    def docker_extra_params = (args.multi_gpu) ? "CI_DOCKER_EXTRA_PARAMS_INIT='--shm-size=4g'" : ''
    sh "${docker_extra_params} ${dockerRun} ${container_type} ${docker_binary} ${docker_args} tests/ci_build/test_python.sh ${mgpu_indicator}"
    if (args.test_rmm) {
      sh "rm -rfv build/ python-package/dist/"
      unstash name: "xgboost_whl_rmm_cuda${args.host_cuda_version}"
      unstash name: "xgboost_cpp_tests_rmm_cuda${args.host_cuda_version}"
      sh "${docker_extra_params} ${dockerRun} ${container_type} ${docker_binary} ${docker_args} tests/ci_build/test_python.sh ${mgpu_indicator} --use-rmm-pool"
    }
    deleteDir()
  }
}

def TestCppGPU(args) {
  def nodeReq = 'linux && mgpu'
  def artifact_cuda_version = (args.artifact_cuda_version) ?: ref_cuda_ver
  node(nodeReq) {
    unstash name: "xgboost_cpp_tests_cuda${artifact_cuda_version}"
    unstash name: 'srcs'
    echo "Test C++, CUDA ${args.host_cuda_version}"
    def container_type = "gpu"
    def docker_binary = "nvidia-docker"
    def docker_args = "--build-arg CUDA_VERSION_ARG=${args.host_cuda_version}"
    sh "${dockerRun} ${container_type} ${docker_binary} ${docker_args} build/testxgboost"
    if (args.test_rmm) {
      sh "rm -rfv build/"
      unstash name: "xgboost_cpp_tests_rmm_cuda${args.host_cuda_version}"
      echo "Test C++, CUDA ${args.host_cuda_version} with RMM"
      container_type = "rmm"
      docker_binary = "nvidia-docker"
      docker_args = "--build-arg CUDA_VERSION_ARG=${args.host_cuda_version}"
      sh """
      ${dockerRun} ${container_type} ${docker_binary} ${docker_args} bash -c "source activate gpu_test && build/testxgboost --use-rmm-pool --gtest_filter=-*DeathTest.*"
      """
    }
    deleteDir()
  }
}

def CrossTestJVMwithJDK(args) {
  node('linux && cpu') {
    unstash name: 'xgboost4j_jar'
    unstash name: 'srcs'
    if (args.spark_version != null) {
      echo "Test XGBoost4J on a machine with JDK ${args.jdk_version}, Spark ${args.spark_version}"
    } else {
      echo "Test XGBoost4J on a machine with JDK ${args.jdk_version}"
    }
    def container_type = "jvm_cross"
    def docker_binary = "docker"
    def spark_arg = (args.spark_version != null) ? "--build-arg SPARK_VERSION=${args.spark_version}" : ""
    def docker_args = "--build-arg JDK_VERSION=${args.jdk_version} ${spark_arg}"
    // Run integration tests only when spark_version is given
    def docker_extra_params = (args.spark_version != null) ? "CI_DOCKER_EXTRA_PARAMS_INIT='-e RUN_INTEGRATION_TEST=1'" : ""
    sh """
    ${docker_extra_params} ${dockerRun} ${container_type} ${docker_binary} ${docker_args} tests/ci_build/test_jvm_cross.sh
    """
    deleteDir()
  }
}

def DeployJVMPackages(args) {
  node('linux && cpu') {
    unstash name: 'srcs'
    if (env.BRANCH_NAME == 'master' || env.BRANCH_NAME.startsWith('release')) {
      echo 'Deploying to xgboost-maven-repo S3 repo...'
      sh """
      ${dockerRun} jvm_gpu_build docker --build-arg CUDA_VERSION_ARG=11.0 tests/ci_build/deploy_jvm_packages.sh ${args.spark_version}
      """
    }
    deleteDir()
  }
}
