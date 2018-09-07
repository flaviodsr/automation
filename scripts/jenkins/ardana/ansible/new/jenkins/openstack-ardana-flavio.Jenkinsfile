/**
 * The openstack-ardana Jenkins Pipeline
 */

pipeline {
  // skip the default checkout, because we want to use a custom path
  options {
    skipDefaultCheckout()
  }

  agent {
    node {
      label 'cloud-ardana-ci'
      customWorkspace ardana_env ? "${JOB_NAME}-${ardana_env}" : "${JOB_NAME}-${BUILD_NUMBER}"
    }
  }

  stages {
    stage('Setup workspace') {
      steps {
        cleanWs()
        script {
          if (ardana_env == '') {
            error("Empty 'ardana_env' parameter value.")
          }
          currentBuild.displayName = "#${BUILD_NUMBER} ${ardana_env}"
        }
        sh '''
          set +x
          git clone $git_automation_repo --branch $git_automation_branch automation-git
        '''
        ansible_playbook(
            playbook: "load-job-params.yml"
        )
      }
    }

    stage('Generate input model') {
      steps {
        ansible_playbook(
            playbook: "generate-input-model.yml",
            extra_vars: "@input.yml"
        )
      }
    }

    stage('Create infra and build package') {
      // abort all stages if one of them fails
      failFast true
      parallel {
        stage('Create infra') {
          steps {
            ansible_playbook(
                playbook: "create-ardana-infra.yml",
                extra_vars: "@input.yml"
            )
          }
        }

        stage('Build test packages') {
          when {
            expression { gerrit_change_ids != '' }
          }
          steps {
            script {
              def slaveJob = build job: 'openstack-ardana-testbuild-gerrit', parameters: [
                string(name: 'gerrit_change_ids', value: "$gerrit_change_ids"),
                string(name: 'develproject', value: "$develproject"),
                string(name: 'homeproject', value: "$homeproject"),
                string(name: 'repository', value: "$repository"),
                string(name: 'git_automation_repo', value: "$git_automation_repo"),
                string(name: 'git_automation_branch', value: "$git_automation_branch")
              ], propagate: true, wait: true

              // Load the environment variables set by the downstream job
              env.test_repository_url = slaveJob.buildVariables.test_repository_url
            }
          }
        }
      } // parallel
    } // stage('parallel stage')

    stage('Bootstrap CLM') {
      steps {
        ansible_playbook(
            playbook: "bootstrap-clm.yml",
            extra_vars: "@input.yml"
        )
      }
    }

    stage('Bootstrap nodes') {
      steps {
        ansible_playbook(
            playbook: "bootstrap-ardana-nodes.yml",
            extra_vars: "@input.yml"
        )
      }
    }

    stage('Deploy cloud') {
        steps {
          ansible_playbook(
              playbook: "deploy-cloud.yml",
              extra_vars: "@input.yml"
          )
        }
      }

    stage('Run tests') {
      // run all stages to the end, even if one of them fails
      failFast false
      parallel {

        stage ('Tempest') {
          when {
            expression { tempest_run_filter != '' }
          }
          steps {
            ansible_playbook(
                playbook: "run-tempest.yml",
                extra_vars: "@input.yml"
            )
          }
          post {
            always {
              junit testResults: '.artifacts/*.xml', allowEmptyResults: true
            }
          }
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: '.artifacts/**/*', allowEmptyArchive: true
      lock(resource: 'cloud-ECP-API') {
        sh '''
          if [ "$cleanup" == "always" ]; then
            cd automation-git/scripts/jenkins/ardana/ansible/new
            ansible-playbook -e heat_action='delete' \
                             -e rc_notify=False \
                             heat-stack.yml
          fi
        '''
      }
    }
    success {
      lock(resource: 'cloud-ECP-API') {
        sh '''
          if [ "$cleanup" == "on success" ]; then
            cd automation-git/scripts/jenkins/ardana/ansible/new
            ansible-playbook -e heat_action='delete' \
                             -e rc_notify=False \
                             heat-stack.yml
          fi
        '''
      }
    }
    failure {
      lock(resource: 'cloud-ECP-API') {
        sh '''
          if [ "$cleanup" == "on failure" ]; then
            cd automation-git/scripts/jenkins/ardana/ansible/new
            ansible-playbook -e heat_action='delete' \
                             -e rc_notify=False \
                             heat-stack.yml
          fi
        '''
      }
    }
  }
}

def ansible_playbook(Map args) {
  sh """
    set +x
    export ANSIBLE_FORCE_COLOR=true
    source /opt/ansible/bin/activate
    cd automation-git/scripts/jenkins/ardana/ansible/new
    ansible-playbook ${args.playbook} -e "${args.extra_vars}"
  """
}
