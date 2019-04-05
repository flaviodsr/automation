/**
 * The openstack-ardana-maintenance-gating Jenkins Pipeline
 */

def ardana_lib = null

pipeline {
  // skip the default checkout, because we want to use a custom path
  options {
    skipDefaultCheckout()
    timestamps()
  }

  agent {
    node {
      label 'cloud-trigger'
      customWorkspace "${JOB_NAME}-${BUILD_NUMBER}"
    }
  }

  stages {
    stage('Setup workspace') {
      steps {
        script {
          sh('''
            git clone https://github.com/flaviodsr/automation.git --branch enable_migrate automation-git
          ''')

          ardana_lib = load "$WORKSPACE/automation-git/jenkins/ci.suse.de/pipelines/openstack-ardana.groovy"
        }
      }
    }

    stage('Trigger jobs') {
      // Do not abort all stages if one of them fails
      failFast false
      parallel {
        stage('es-kvm-virtual') {
          steps {
            script {
              def slaveJob = ardana_lib.trigger_build("flavio-ardana9-es-kvm-virtual", [], false)
            }
          }
        }

        stage('es-kvm-bm') {
          steps {
            script {
              def slaveJob = ardana_lib.trigger_build("flavio-ardana9-es-kvm-bm", [], false)
            }
          }
        }
      }
    }
  }
  post{
    cleanup {
      cleanWs()
    }
  }
}
