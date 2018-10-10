/**
 * The openstack-ardana-tempest Jenkins Pipeline
 *
 * This job runs tempest on a pre-deployed CLM cloud.
 */

pipeline {

  options {
    // skip the default checkout, because we want to use a custom path
    skipDefaultCheckout()
    timestamps()
  }

  agent {
    node {
      label reuse_node ? reuse_node : "cloud-ardana-ci"
      customWorkspace "${JOB_NAME}-${BUILD_NUMBER}"
    }
  }

  stages {
    stage('Setup workspace') {
      steps {
        script {
          echo "Done"
        }
      }
    }

    stage('Run Tempest') {
      steps {
        script {
          sh('exit 1')
        }
      }
    }

    stage('Run QA tests') {
      when {
        // For extended-choice parameter we also need to check if the variable
        // is defined
        expression { env.qa_test_list != null && qa_test_list != '' }
      }
      steps {
        script {
          echo "Done"
        }
      }
    }

  }
}
