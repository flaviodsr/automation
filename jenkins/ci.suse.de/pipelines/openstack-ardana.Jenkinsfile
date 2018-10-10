/**
 * The openstack-ardana Jenkins Pipeline
 */

pipeline {
  options {
    // skip the default checkout, because we want to use a custom path
    skipDefaultCheckout()
    timestamps()
    // reserve a resource if instructed to do so, otherwise use a dummy resource
    // and a zero quantity to fool Jenkins into thinking it reserved a resource when in fact it didn't
    lock(label: reserve_env == 'true' ? ardana_env:'dummy-resource',
         variable: 'reserved_env',
         quantity: reserve_env == 'true' ? 1:0 )
  }

  agent {
    node {
      label 'cloud-ardana-ci'
      customWorkspace "${JOB_NAME}-${BUILD_NUMBER}"
    }
  }

  stages {
    stage('Setup workspace') {
      steps {
        script {
          currentBuild.displayName = "#${BUILD_NUMBER}: ${ardana_env}"
          env.cloud_type = "virtual"
          echo "Done"
        }
      }
    }

    stage('Prepare infra and build package(s)') {
      // abort all stages if one of them fails
      failFast true
      parallel {
        stage('Prepare BM cloud') {
          when {
            expression { cloud_type == 'physical' }
          }
          steps {
            script {
              echo "Done"
            }
          }
        }

        stage('Prepare virtual cloud') {
          when {
            expression { cloud_type == 'virtual' }
          }
          steps {
            script {
              echo "Done"
            }
          }
        }

        stage('Build test packages') {
          when {
            expression { gerrit_change_ids != '' }
          }
          steps {
            script {
              echo "Done"
            }
          }
        }
      } // parallel
    } // stage('parallel stage')

    stage('Bootstrap CLM') {
      steps {
        script {
          echo "Done"
        }
      }
    }

    stage('Bootstrap nodes') {
      failFast true
      parallel {
        stage('Bootstrap BM nodes') {
          when {
            expression { cloud_type == 'physical' }
          }
          steps{
            script {
              echo "Done"
            }
          }
        }

        stage('Bootstrap virtual nodes') {
          when {
            expression { cloud_type == 'virtual' }
          }
          steps{
            script {
              echo "Done"
            }
          }
        }
      }
    }

    stage('Deploy cloud') {
      steps {
        script {
          echo "Done"
        }
      }
    }

    stage('Start tests') {
      when {
        expression { tempest_run_filter != '' }
      }
      steps {
        script {
          slaveJob = null
          catchError {
            stage('Run tests') {
              script {
                slaveJob = build job: 'openstack-ardana-tempest', parameters: [
                  string(name: 'ardana_env', value: "$ardana_env"),
                  string(name: 'tempest_run_filter', value: "$tempest_run_filter"),
                  string(name: 'rc_notify', value: "$rc_notify"),
                  string(name: 'git_automation_repo', value: "$git_automation_repo"),
                  string(name: 'git_automation_branch', value: "$git_automation_branch"),
                  string(name: 'reuse_node', value: "${NODE_NAME}")
                ], propagate: true, wait: true
              }
            }
          }
          def jobResult = slaveJob.getResult()
          def jobUrl = slaveJob.buildVariables.blue_ocean_buildurl
          def jobMsg = "Build ${jobUrl} completed with: ${jobResult}"
          echo jobMsg
          if (jobResult != 'SUCCESS') {
             currentBuild.result = "SUCCESS"
             // error(jobMsg)
          }
        }
      }
    }

    stage('Update cloud') {
      steps {
        script {
          echo "Done"
        }
      }
    }

    stage('Deploy CaaSP') {
      when {
        expression { want_caasp == 'true' }
      }
      steps {
        sh('''
          cd $SHARED_WORKSPACE
          source automation-git/scripts/jenkins/ardana/jenkins-helper.sh
          ansible_playbook deploy-caasp.yml -e @input.yml
        ''')
      }
    }
  }

  post {
    always {
      script{
        archiveArtifacts artifacts: ".artifacts/**/*", allowEmptyArchive: true
        if ( tempest_run_filter != '' ) {
          junit testResults: ".artifacts/*.xml", allowEmptyResults: true
        }
      }
      cleanWs()
      script{
        if (cleanup == "always" && cloud_type == "virtual") {
          def slaveJob = build job: 'openstack-ardana-heat', parameters: [
            string(name: 'ardana_env', value: "$ardana_env"),
            string(name: 'heat_action', value: "delete"),
            string(name: 'git_automation_repo', value: "$git_automation_repo"),
            string(name: 'git_automation_branch', value: "$git_automation_branch")
          ], propagate: false, wait: false
        }
      }
    }
    success {
      script {
        if (cleanup == "on success" && cloud_type == "virtual") {
          def slaveJob = build job: 'openstack-ardana-heat', parameters: [
            string(name: 'ardana_env', value: "$ardana_env"),
            string(name: 'heat_action', value: "delete"),
            string(name: 'git_automation_repo', value: "$git_automation_repo"),
            string(name: 'git_automation_branch', value: "$git_automation_branch")
          ], propagate: false, wait: false
        }
      }
      sh '''
        if [ -n "$github_pr" ] ; then
          cd $SHARED_WORKSPACE
          exec automation-git/scripts/ardana/pr-success.sh
        fi
      '''
    }
    failure {
      script {
        if (cleanup == "on failure" && cloud_type == "virtual") {
          def slaveJob = build job: 'openstack-ardana-heat', parameters: [
            string(name: 'ardana_env', value: "$ardana_env"),
            string(name: 'heat_action', value: "delete"),
            string(name: 'git_automation_repo', value: "$git_automation_repo"),
            string(name: 'git_automation_branch', value: "$git_automation_branch")
          ], propagate: false, wait: false
        }
      }
      sh '''
        if [ -n "$github_pr" ] ; then
          cd $SHARED_WORKSPACE
          exec automation-git/scripts/ardana/pr-failure.sh
        fi
      '''
    }
  }
}
