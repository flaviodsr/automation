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
          env.cloud_type = "virtual"
          if (ardana_env == '') {
            error("Empty 'ardana_env' parameter value.")
          }
          if ( ardana_env.startsWith("qe") || ardana_env.startsWith("qa") ) {
              env.cloud_type = "physical"
          }
          currentBuild.displayName = "#${BUILD_NUMBER} ${ardana_env}"
        }
        sh('git clone $git_automation_repo --branch $git_automation_branch automation-git')
        ansible_playbook(
          playbook: "load-job-params.yml"
        )
        ansible_playbook(
          playbook: "notify-rc-pcloud.yml",
          extra_vars: "-e @input.yml"
        )
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
              def slaveJob = build job: 'openstack-ardana-pcloud-flavio', parameters: [
                string(name: 'ardana_env', value: "$ardana_env"),
                string(name: 'scenario_name', value: "$scenario_name"),
                string(name: 'clm_model', value: "$clm_model"),
                string(name: 'controllers', value: "$controllers"),
                string(name: 'sles_computes', value: "$sles_computes"),
                string(name: 'rhel_computes', value: "$rhel_computes"),
                string(name: 'rc_notify', value: "$rc_notify"),
                string(name: 'git_automation_repo', value: "$git_automation_repo"),
                string(name: 'git_automation_branch', value: "$git_automation_branch"),
                string(name: 'reuse_node', value: "${NODE_NAME}"),
                string(name: 'reuse_workspace', value: "${WORKSPACE}")
              ], propagate: true, wait: true
            }
          }
        }

        stage('Prepare virtual cloud') {
          when {
            expression { cloud_type == 'virtual' }
          }
          steps {
            script {
              def slaveJob = build job: 'openstack-ardana-vcloud-flavio', parameters: [
                string(name: 'ardana_env', value: "$ardana_env"),
                string(name: 'git_input_model_branch', value: "$git_input_model_branch"),
                string(name: 'git_input_model_path', value: "$git_input_model_path"),
                string(name: 'model', value: "$model"),
                string(name: 'scenario_name', value: "$scenario_name"),
                string(name: 'clm_model', value: "$clm_model"),
                string(name: 'controllers', value: "$controllers"),
                string(name: 'sles_computes', value: "$sles_computes"),
                string(name: 'rhel_computes', value: "$rhel_computes"),
                string(name: 'rc_notify', value: "$rc_notify"),
                string(name: 'git_automation_repo', value: "$git_automation_repo"),
                string(name: 'git_automation_branch', value: "$git_automation_branch"),
                string(name: 'reuse_node', value: "${NODE_NAME}"),
                string(name: 'reuse_workspace', value: "${WORKSPACE}")
              ], propagate: true, wait: true
            }
          }
        }

        stage('Build test packages') {
          when {
            expression { gerrit_change_ids != '' }
          }
          steps {
            script {
              def slaveJob = build job: 'openstack-ardana-testbuild-gerrit-flavio', parameters: [
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
        script {
          def slaveJob = build job: 'openstack-ardana-bootstrap-clm-flavio', parameters: [
            string(name: 'ardana_env', value: "$ardana_env"),
            string(name: 'cloudsource', value: "$cloudsource"),
            string(name: 'cloud_maint_updates', value: "$cloud_maint_updates"),
            string(name: 'sles_maint_updates', value: "$sles_maint_updates"),
            string(name: 'extra_repos', value: "${env.test_repository_url ?: extra_repos}"),
            string(name: 'rc_notify', value: "$rc_notify"),
            string(name: 'git_automation_repo', value: "$git_automation_repo"),
            string(name: 'git_automation_branch', value: "$git_automation_branch"),
            string(name: 'reuse_node', value: "${NODE_NAME}"),
            string(name: 'reuse_workspace', value: "${WORKSPACE}")
          ], propagate: true, wait: true
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
            ansible_playbook(
              playbook: "bootstrap-pcloud-nodes.yml",
              extra_vars: "-e @input.yml"
            )
          }
        }

        stage('Bootstrap virtual nodes') {
          when {
            expression { cloud_type == 'virtual' }
          }
          steps{
            ansible_playbook(
              playbook: "bootstrap-vcloud-nodes.yml",
              extra_vars: "-e @input.yml"
            )
          }
        }
      }
    }

    stage('Deploy cloud') {
        steps {
          ansible_playbook(
              playbook: "deploy-cloud.yml",
              extra_vars: "-e @input.yml"
          )
        }
      }

    stage('Run tests') {
      failFast false
      parallel {
        stage ('Tempest') {
          when {
            expression { tempest_run_filter != '' }
          }
          steps {
            script {
              def slaveJob = build job: 'openstack-ardana-tempest-flavio', parameters: [
                string(name: 'ardana_env', value: "$ardana_env"),
                string(name: 'tempest_run_filter', value: "$tempest_run_filter"),
                string(name: 'rc_notify', value: "$rc_notify"),
                string(name: 'git_automation_repo', value: "$git_automation_repo"),
                string(name: 'git_automation_branch', value: "$git_automation_branch"),
                string(name: 'reuse_node', value: "${NODE_NAME}"),
                string(name: 'reuse_workspace', value: "${WORKSPACE}")
              ], propagate: true, wait: true
            }
          }
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: '.artifacts/**/*', allowEmptyArchive: true
      script{
        if (cleanup == "always") {
          lock(resource: 'cloud-ECP-API') {
            ansible_playbook(
              playbook: "heat-stack.yml",
              extra_vars: "-e @input.yml -e heat_action=delete"
            )
          }
        }
      }
    }
    success {
      script {
        if (cleanup == "on success") {
          lock(resource: 'cloud-ECP-API') {
            ansible_playbook(
              playbook: "heat-stack.yml",
              extra_vars: "-e @input.yml -e heat_action=delete"
            )
          }
        }
      }
    }
    failure {
      script {
        if (cleanup == "on failure") {
          lock(resource: 'cloud-ECP-API') {
            ansible_playbook(
              playbook: "heat-stack.yml",
              extra_vars: "-e @input.yml -e heat_action=delete"
            )
          }
        }
      }
    }
  }
}

def ansible_playbook(Map args) {
  sh """
    set +x
    export ANSIBLE_FORCE_COLOR=true
    source /opt/ansible/bin/activate
    cd automation-git/scripts/jenkins/ardana/ansible
    ansible-playbook ${args.playbook} ${args.extra_vars ?: ""}
  """
}
