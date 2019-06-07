/**
 * The openstack-ardana Jenkins Pipeline
 */

def ardana_lib = null

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
          // Set this variable to be used by upstream builds
          env.blue_ocean_buildurl = env.RUN_DISPLAY_URL
          env.cloud_type = "virtual"
          if (ardana_env == '') {
            error("Empty 'ardana_env' parameter value.")
          }
          if (updates_test_enabled == 'true' && maint_updates != '') {
            error("Maintenance updates and test-updates cannot be applied at the same time.")
          }
          if (reserve_env == 'true') {
            echo "Reserved resource: " + env.reserved_env
            if (env.reserved_env && reserved_env != null) {
              env.ardana_env = reserved_env
            } else {
              error("Jenkins bug (JENKINS-52638): couldn't reserve a resource with label $ardana_env")
            }
          }
          currentBuild.displayName = "#${BUILD_NUMBER}: ${ardana_env}"
          if ( ardana_env.startsWith("qe") || ardana_env.startsWith("pcloud") ) {
              env.cloud_type = "physical"
          }
          // Parameters of the type 'extended-choice' are set to null when the job
          // is automatically triggered and its value is set to ''. So, we need to set
          // it to '' to be able to pass it as a parameter to downstream jobs.
          if (env.tempest_filter_list == null) {
            env.tempest_filter_list = ''
          }
          if (env.qa_test_list == null) {
            env.qa_test_list = ''
          }
          sh('''
            git clone $git_automation_repo --branch $git_automation_branch automation-git
            cd automation-git

            if [ -n "$github_pr" ] ; then
              scripts/jenkins/ardana/pr-update.sh
            fi
          ''')
          ardana_lib = load "$WORKSPACE/automation-git/jenkins/ci.suse.de/pipelines/openstack-ardana.groovy"
        }
      }
    }
    stage ('Run jobs') {
      steps {
        script {
          def map = [:]
          map[8] = ['ardana', 'crowbar']
          map[9] = ['ardana']
          parallel map.collectEntries {
              k, v -> ["Cloud-${k}" : ardana_lib.generate_version_stages(k, v)]
          }
        }
      }
    }
  }
}
