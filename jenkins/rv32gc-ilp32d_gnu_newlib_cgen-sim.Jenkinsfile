node ('buildnode') {
  stage('Cleanup') {
    deleteDir()
  }

  stage('Checkout') {
    checkout scm
    dir('binutils-gdb') {
      git url: 'https://sourceware.org/git/binutils-gdb.git', branch: 'master'
    }
    dir('binutils-gdb-cgen') {
      git url: 'https://github.com/embecosm/riscv-binutils-gdb.git', branch: 'wip-cgen'
    }
    dir('gcc') {
      checkout([$class: 'GitSCM',
                branches: [[name: '*/master']],
                doGenerateSubmoduleConfigurations: false,
                extensions: [[$class: 'CloneOption',
                              noTags: false,
                              reference: '',
                              shallow: true]],
                submoduleCfg: [],
                userRemoteConfigs:
                [[url: 'https://github.com/gcc-mirror/gcc.git']]])
    }
    dir('newlib') {
      git url: 'git://sourceware.org/git/newlib-cygwin.git', branch: 'master'
    }
  }

  stage('Build Binutils and GCC') {
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
          dir ('build/binutils-gdb-cgen') {
            sh script: '''${WORKSPACE}/binutils-gdb-cgen/configure \
                          --target=riscv32-unknown-elf             \
                          --prefix=${WORKSPACE}/install            \
                          --disable-gdb                            \
                          --enable-sim                             \
                          --disable-werror                         \
                          > ${WORKSPACE}/build/sim-config.log 2>&1'''
            sh script: '''make -j$(nproc) all-sim > ${WORKSPACE}/build/sim-build.log 2>&1'''
            sh script: '''make install-sim > ${WORKSPACE}/build/sim-install.log 2>&1'''
          }
          dir ('build/gcc') {
            sh script: '''${WORKSPACE}/gcc/configure    \
                          --target=riscv32-unknown-elf  \
                          --prefix=${WORKSPACE}/install \
                          --without-newlib              \
                          --without-headers             \
                          --enable-languages=c          \
                          --disable-werror              \
                          --disable-libatomic           \
                          --disable-libmudflap          \
                          --disable-libssp              \
                          --disable-quadmath            \
                          --disable-libgomp             \
                          --disable-nls                 \
                          --disable-bootstrap           \
                          --with-arch=rv32gc            \
                          --with-abi=ilp32d             \
                          > ${WORKSPACE}/build/gcc-config.log 2>&1'''
            sh script: '''make -j$(nproc) > ${WORKSPACE}/build/gcc-build.log 2>&1'''
            sh script: '''make install > ${WORKSPACE}/build/gcc-install.log 2>&1'''
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

  stage('GCC regression') {
    timeout(120) {
      try {
        docker.image('embecosm/buildenv').inside {
          sh script: '''./run-tests.py                        \
                        --triple riscv32-unknown-elf          \
                        --arch rv32gc                         \
                        --abi ilp32d                          \
                        --tools-dir ${WORKSPACE}/install/bin  \
                        --cc riscv32-unknown-elf-gcc          \
                        --gcc-source ${WORKSPACE}/gcc         \
                        --sim-command riscv32-unknown-elf-run \
                        > check-gcc.log 2>&1'''
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
