node ('buildnode') {
  stage('Cleanup') {
    deleteDir()
  }

  stage('Checkout') {
    checkout scm
    dir('binutils-gdb') {
      git url: 'https://sourceware.org/git/binutils-gdb.git', branch: 'master'
    }
    dir('llvm') {
      git url: 'https://github.com/llvm/llvm-project.git', branch: 'master'
    }
    dir('newlib') {
      git url: 'git://sourceware.org/git/newlib-cygwin.git', branch: 'master'
    }
    dir('qemu') {
      git url: 'https://github.com/qemu/qemu.git', branch: 'master'
    }
    dir('gcc-for-llvm-testing') {
      git url: 'https://github.com/embecosm/gcc-for-llvm-testing', branch: 'llvm-testing'
    }
  }

  stage('Build Binutils and Clang/LLVM') {
    timeout(60) {
      try {
        docker.image('embecosm/buildenv').inside {
          dir ('build/binutils-gdb') {
            sh script: '''${WORKSPACE}/binutils-gdb/configure  \
                          --target=riscv32-unknown-elf         \
                          --prefix=${WORKSPACE}/install        \
                          --disable-werror                     \
                          > ${WORKSPACE}/build/binutils-config.log 2>&1'''
            sh script: '''make -j$(nproc) > ${WORKSPACE}/build/binutils-build.log 2>&1'''
            sh script: '''make install > ${WORKSPACE}/build/binutils-install.log 2>&1'''
          }
          dir ('build/llvm') {
            sh script: '''cmake ${WORKSPACE}/llvm/llvm                \
                          -G "Ninja"                                  \
                          -DCMAKE_INSTALL_PREFIX=${WORKSPACE}/install \
                          -DCMAKE_BUILD_TYPE=Release                  \
                          -DLLVM_ENABLE_ASSERTIONS=On                 \
                          -DLLVM_TARGETS_TO_BUILD="X86;RISCV"         \
                          -DLLVM_USE_LINKER=gold                      \
                          -DLLVM_ENABLE_PROJECTS='clang'              \
                          -DBUILD_SHARED_LIBS=ON                      \
                          -DLLVM_BINUTILS_INCDIR=${WORKSPACE}/binutils-gdb/include \
                          > ${WORKSPACE}/build/llvm-config.log 2>&1'''
            sh script: '''cmake --build . > ${WORKSPACE}/build/llvm-build.log 2>&1'''
            sh script: '''cmake --build . --target install > ${WORKSPACE}/build/llvm-install.log 2>&1'''
          }
        }
      }
      finally {
        archiveArtifacts allowEmptyArchive: true, fingerprint: true,
            artifacts: 'build/*.log'
      }
    }
  }

  stage('Build Newlib') {
    timeout(15) {
      try {
        docker.image('embecosm/buildenv').inside {
          dir ('build/newlib') {
            sh script: '''PATH="${WORKSPACE}/install/bin:$PATH" \
                          CC_FOR_TARGET=clang                   \
                          CFLAGS_FOR_TARGET="-target riscv32-unknown-elf -march=rv32gc -mabi=ilp32d -O0 -g3" \
                          ${WORKSPACE}/newlib/configure         \
                          --target=riscv32-unknown-elf          \
                          --prefix=${WORKSPACE}/install         \
                          --with-arch=rv32gc                    \
                          > ${WORKSPACE}/build/newlib-config.log 2>&1'''
            sh script: '''PATH="${WORKSPACE}/install/bin:$PATH" \
                          make -j$(nproc)                       \
                          > ${WORKSPACE}/build/newlib-build.log 2>&1'''
            sh script: '''PATH="${WORKSPACE}/install/bin:$PATH" \
                          make install                          \
                          > ${WORKSPACE}/build/newlib-install.log 2>&1'''
          }
        }
      }
      finally {
        archiveArtifacts allowEmptyArchive: true, fingerprint: true,
            artifacts: 'build/*.log'
      }
    }
  }

  stage('Build Compiler-RT') {
    timeout(15) {
      try {
        docker.image('embecosm/buildenv').inside {
          dir ('build/compiler-rt') {
            sh script: '''cmake ${WORKSPACE}/llvm/compiler-rt                     \
                          -G "Ninja"                                              \
                          -DCMAKE_INSTALL_PREFIX=${WORKSPACE}/install/riscv32-unknown-elf \
                          -DCOMPILER_RT_BUILD_BUILTINS=ON                         \
                          -DCOMPILER_RT_BUILD_SANITIZERS=OFF                      \
                          -DCOMPILER_RT_BUILD_XRAY=OFF                            \
                          -DCOMPILER_RT_BUILD_LIBFUZZER=OFF                       \
                          -DCOMPILER_RT_BUILD_PROFILE=OFF                         \
                          -DCMAKE_C_COMPILER=${WORKSPACE}/install/bin/clang       \
                          -DCMAKE_AR=${WORKSPACE}/install/bin/llvm-ar             \
                          -DCMAKE_NM=${WORKSPACE}/install/bin/llvm-nm             \
                          -DCMAKE_RANLIB=${WORKSPACE}/install/bin/llvm-ranlib     \
                          -DCMAKE_C_COMPILER_TARGET="riscv32-unknown-elf"         \
                          -DCMAKE_ASM_COMPILER_TARGET="riscv32-unknown-elf"       \
                          -DCOMPILER_RT_DEFAULT_TARGET_ONLY=ON                    \
                          -DLLVM_CONFIG_PATH=${WORKSPACE}/install/bin/llvm-config \
                          -DCMAKE_C_FLAGS="-march=rv32gc -mabi=ilp32d"            \
                          -DCMAKE_ASM_FLAGS="-march=rv32gc -mabi=ilp32d"          \
                          -DCMAKE_EXE_LINKER_FLAGS="-nostartfiles -nostdlib"      \
                          -DCOMPILER_RT_BAREMETAL_BUILD=ON                        \
                          -DCOMPILER_RT_OS_DIR=""                                 \
                          > ${WORKSPACE}/build/compiler-rt-config.log 2>&1'''
            sh script: '''cmake --build . \
                          > ${WORKSPACE}/build/compiler-rt-build.log 2>&1'''
            sh script: '''cmake --build . --target install \
                          > ${WORKSPACE}/build/compiler-rt-install.log 2>&1'''
          }
        }
      finally {
        archiveArtifacts allowEmptyArchive: true, fingerprint: true,
            artifacts: 'build/*.log'
      }
    }
  }

  stage('Build QEMU Linux User Mode') {
    timeout(15) {
      try {
        docker.image('embecosm/buildenv').inside {
          dir ('build/qemu') {
            sh script: '''${WORKSPACE}/qemu/configure      \
                          --target-list=riscv32-linux-user \
                          --prefix=${WORKSPACE}/install    \
                          --static                         \
                          --disable-gtk                    \
                          --disable-vte                    \
                          > ../qemu-config.log 2>&1'''
            sh script: '''make -j$(nproc) > ${WORKSPACE}/build/qemu-build.log 2>&1'''
            sh script: '''make install > ${WORKSPACE}/build/qemu-install.log 2>&1'''
          }
        }
      }
      finally {
        archiveArtifacts allowEmptyArchive: true, fingerprint: true,
            artifacts: 'build/*.log'
      }
    }
  }

  stage('GCC regression') {
    timeout(120) {
      try {
        docker.image('embecosm/buildenv').inside {
          sh script: '''./run-tests.py                                 \
                        --triple riscv32-unknown-elf                   \
                        --arch rv32gc                                  \
                        --abi ilp32d                                   \
                        --tools-dir ${WORKSPACE}/install/bin           \
                        --cc clang                                     \
                        --cflags "-target riscv32-unknown-elf"         \
                        --gcc-source ${WORKSPACE}/gcc-for-llvm-testing \
                        --sim-command qemu-riscv32                     \
                        > ${WORKSPACE}/check-gcc.log 2>&1'''
        }
      }
      catch (Exception e) {}
      finally {
        archiveArtifacts allowEmptyArchive: true, fingerprint: true,
            artifacts: 'check-gcc.log, test-output/gcc.log, test-output/gcc.sum'
      }
    }
  }
}
