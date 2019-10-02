node ('buildnode') {
  stage('Cleanup') {
    deleteDir()
  }

  stage('Checkout') {
    checkout scm
    dir('binutils-gdb') {
      git url: 'https://sourceware.org/git/binutils-gdb.git', branch: 'master'
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
    dir('noname-gdbserver') {
      git url: 'https://github.com/embecosm/noname-gdbserver.git', branch: 'master'
      dir('targets') {
        checkout([$class: 'GitSCM',
                  branches: [[name: '*/master']],
                  doGenerateSubmoduleConfigurations: false,
                  extensions: [[$class: 'SubmoduleOption',
                                disableSubmodules: false,
                                parentCredentials: false,
                                recursiveSubmodules: true,
                                reference: '',
                                trackingSubmodules: false]],
                  submoduleCfg: [],
                  userRemoteConfigs:
                  [[url: 'https://github.com/embecosm/ri5cy-gdbserver-target.git']]])
      }
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
          dir ('build/gcc') {
            sh script: '''${WORKSPACE}/gcc/configure    \
                          --target=riscv32-unknown-elf  \
                          --prefix=${WORKSPACE}/install \
                          --with-newlib                 \
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
                          --with-arch=rv32imac          \
                          --with-abi=ilp32              \
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
                          --with-arch=rv32imac                  \
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

  stage('Build NoName GDBServer and ri5cy target') {
    timeout(15) {
      try {
        docker.image('embecosm/buildenv').inside {
          dir('build/noname-gdbserver') {
            sh script: '''cmake ${WORKSPACE}/noname-gdbserver \
                          -DCMAKE_INSTALL_PREFIX=${WORKSPACE}/install \
                          > ${WORKSPACE}/build/noname-gdbserver-config.log 2>&1'''
            sh script: '''cmake --build . \
                          > ${WORKSPACE}/build/noname-gdbserver-build.log 2>&1'''
            sh script: '''cmake --build . --target install \
                          > ${WORKSPACE}/build/noname-gdbserver-install.log 2>&1'''
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
                        --board riscv-gdbserver               \
                        --arch rv32imac                       \
                        --abi ilp32                           \
                        --tools-dir ${WORKSPACE}/install/bin  \
                        --cc riscv32-unknown-elf-gcc          \
                        --gcc-source ${WORKSPACE}/gcc         \
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
