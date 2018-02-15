pipeline {
  agent none
  stages {

    stage('Build') {
      steps {
        script {
          def builds = [:]

          def docker_base = "px4io/px4-dev-base:2017-12-30"
          def docker_nuttx = "px4io/px4-dev-nuttx:2017-12-30"
          def docker_rpi = "px4io/px4-dev-raspi:2017-12-30"
          def docker_armhf = "px4io/px4-dev-armhf:2017-12-30"
          def docker_arch = "px4io/px4-dev-base-archlinux:2017-12-30"

          builds["px4fmu-v2"] = createBuildNode(docker_nuttx, "nuttx_px4fmu-v2_default")

          // nuttx default targets that are archived and uploaded to s3
          for (def option in ["px4fmu-v3", "px4fmu-v4", "px4fmu-v4pro", "px4fmu-v5", "aerofc-v1"]) {
            def node_name = "${option}"
            builds[node_name] = createBuildNode(docker_nuttx, "nuttx_${node_name}_default")
          }

          // nuttx default targets that are archived and uploaded to s3
          for (def option in ["aerocore2", "auav-x21", "crazyflie", "mindpx-v2", "nxphlite-v3", "tap-v1"]) {
            def node_name = "${option}"
            builds[node_name] = createBuildNode(docker_nuttx, "nuttx_${node_name}_default")
          }

          // other nuttx default targets
          for (def option in ["px4-same70xplained-v1", "px4-stm32f4discovery", "px4cannode-v1", "px4esc-v1", "px4nucleoF767ZI-v1", "s2740vc-v1"]) {
            def node_name = "${option}"
            builds[node_name] = createBuildNode(docker_nuttx, "nuttx_${node_name}_default")
          }

          builds["posix_sitl"] = createBuildNode(docker_base, "posix_sitl_default")
          builds["posix_sitl_default (GCC 7)"] = createBuildNode(docker_arch, "posix_sitl_default")

          builds["rpi"] = createBuildNode(docker_rpi, "posix_rpi_cross")
          builds["bebop"] = createBuildNode(docker_rpi, "posix_bebop_default")

          builds["ocpoc"] = createBuildNode(docker_armhf, "posix_ocpoc_ubuntu")

          //builds["snapdragon"] = createBuildNode(docker_snapdragon, "eagle_default")

          parallel builds
        } // script
      } // steps
    } // stage Builds

    stage('Test') {
      parallel {

        stage('check style') {
          agent {
            docker {
              image 'px4io/px4-dev-base:2017-12-30'
            }
          }
          steps {
            sh 'export'
            sh 'make distclean'
            sh 'make check_format'
          }
        }

        stage('clang analyzer') {
          agent {
            docker {
              image 'px4io/px4-dev-clang:2017-12-30'
              args '-e CCACHE_BASEDIR=$WORKSPACE -v ${CCACHE_DIR}:${CCACHE_DIR}:rw'
            }
          }
          steps {
            sh 'export'
            sh 'make distclean'
            sh 'make scan-build'
            // publish html
            publishHTML target: [
              reportTitles: 'clang static analyzer',
              allowMissing: false,
              alwaysLinkToLastBuild: true,
              keepAll: true,
              reportDir: 'build/scan-build/report_latest',
              reportFiles: '*',
              reportName: 'Clang Static Analyzer'
            ]
          }
          when {
            anyOf {
              branch 'master'
              branch 'beta'
              branch 'stable'
            }
          }
        }

        stage('clang tidy') {
          agent {
            docker {
              image 'px4io/px4-dev-clang:2017-12-30'
              args '-e CCACHE_BASEDIR=$WORKSPACE -v ${CCACHE_DIR}:${CCACHE_DIR}:rw'
            }
          }
          steps {
            sh 'export'
            sh 'make distclean'
            sh 'make clang-tidy-quiet'
          }
        }

        stage('cppcheck') {
          agent {
            docker {
              image 'px4io/px4-dev-base:ubuntu17.10'
              args '-e CCACHE_BASEDIR=$WORKSPACE -v ${CCACHE_DIR}:${CCACHE_DIR}:rw'
            }
          }
          steps {
            sh 'export'
            sh 'make distclean'
            sh 'make cppcheck'
            // publish html
            publishHTML target: [
              reportTitles: 'Cppcheck',
              allowMissing: false,
              alwaysLinkToLastBuild: true,
              keepAll: true,
              reportDir: 'build/cppcheck/',
              reportFiles: '*',
              reportName: 'Cppcheck'
            ]
          }
          when {
            anyOf {
              branch 'master'
              branch 'beta'
              branch 'stable'
            }
          }
        }

        stage('tests') {
          agent {
            docker {
              image 'px4io/px4-dev-base:2017-12-30'
              args '-e CCACHE_BASEDIR=$WORKSPACE -v ${CCACHE_DIR}:${CCACHE_DIR}:rw'
            }
          }
          steps {
            sh 'export'
            sh 'make distclean'
            sh 'make posix_sitl_default test_results_junit'
            junit 'build/posix_sitl_default/JUnitTestResults.xml'
          }
        }

      } // steps
    } // stage Builds

    stage('Generate Metadata') {

      parallel {

        stage('airframe') {
          agent {
            docker { image 'px4io/px4-dev-base:2017-12-30' }
          }
          steps {
            sh 'export'
            sh 'make distclean'
            sh 'make airframe_metadata'
            archiveArtifacts(artifacts: 'airframes.md, airframes.xml')
          }
        }

        stage('parameter') {
          agent {
            docker { image 'px4io/px4-dev-base:2017-12-30' }
          }
          steps {
            sh 'export'
            sh 'make distclean'
            sh 'make parameters_metadata'
            archiveArtifacts(artifacts: 'parameters.md, parameters.xml')
          }
        }

        stage('module') {
          agent {
            docker { image 'px4io/px4-dev-base:2017-12-30' }
          }
          steps {
            sh 'export'
            sh 'make distclean'
            sh 'make module_documentation'
            archiveArtifacts(artifacts: 'modules/*.md')
          }
        }

        stage('uorb graphs') {
          agent {
            docker {
              image 'px4io/px4-dev-nuttx:2017-12-30'
              args '-e CCACHE_BASEDIR=$WORKSPACE -v ${CCACHE_DIR}:${CCACHE_DIR}:rw'
            }
          }
          steps {
            sh 'export'
            sh 'make distclean'
            sh 'make uorb_graphs'
            archiveArtifacts(artifacts: 'Tools/uorb_graph/graph_sitl.json')
          }
        }
      }
    }
  }
  environment {
    CCACHE_DIR = '/tmp/ccache'
    CI = true
  }
  options {
    buildDiscarder(logRotator(numToKeepStr: '10'))
    timeout(time: 60, unit: 'MINUTES')
  }
}

def createBuildNode(String docker_repo, String target) {
  return {
    node {
      docker.image(docker_repo).inside('-e CCACHE_BASEDIR=$WORKSPACE -v ${CCACHE_DIR}:${CCACHE_DIR}:rw') {
        stage(target) {
          sh('export')
          checkout scm
          sh('make distclean')
          sh('git fetch --tags')
          sh('ccache -z')
          sh('make ' + target)
          sh('ccache -s')
          sh('make sizes')
          archiveArtifacts(artifacts: 'build/*/*.px4', fingerprint: true)
        }
      }
    }
  }
}

def createROSMissionTest(String test_file, String mission_file, String vehicle_model) {
  return {
    node {
      docker.image('px4io/px4-dev-ros:2017-12-30').inside('-e CCACHE_BASEDIR=$WORKSPACE -v ${CCACHE_DIR}:${CCACHE_DIR}:rw') {
        stage(target) {
          sh('export')
          checkout scm
          sh('make distclean')
          sh('rm -rf .ros; rm -rf .gazebo')
          sh('git fetch --tags')
          sh('ccache -z')
          sh('make posix_sitl_default')
          sh('ccache -s')
          sh('ccache -z')
          sh('make posix_sitl_default sitl_gazebo')
          sh('ccache -s')
          sh('./test/rostest_px4_run.sh ' + test_file + ' mission:=' + mission_file + ' vehicle:=' + vehicle_model)
          sh('./Tools/upload_log.py -q --description "ROS mission test:' + mission_file + ' ${CHANGE_ID}" --feedback "${CHANGE_TITLE} - ${CHANGE_URL}" --source CI .ros/rootfs/fs/microsd/log/*/*.ulg')
          archiveArtifacts(artifacts: '**/*.ulg')
        }
      }
    }
  }
}
