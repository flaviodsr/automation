/**
 * The openstack-ccp Jenkins Pipeline
 */

pipeline {
  options {
    // skip the default checkout, because we want to use a custom path
    skipDefaultCheckout()
    timestamps()
    timeout(time: 30, unit: 'MINUTES', activity: true)
  }

  agent {
    node {
      label 'cloud-ccp-ci'
      customWorkspace "workspace/${PREFIX}-${JOB_NAME}-${BUILD_NUMBER}"
    }
  }

  environment {
    INTERNAL_SUBNET         = "${PREFIX}-subnet"
    ANSIBLE_RUNNER_DIR      = "${WORKSPACE}/suse-osh-deploy"
    SOCOK8S_VENV            = "${HOME}/.socok8svenv"
    USE_ARA                 = "True"
    ANSIBLE_FORCE_COLOR     = "True"
    ANSIBLE_STDOUT_CALLBACK = "yaml"
  }

  stages {
    stage('Setup workspace') {
      steps {
        script {
          hosts = [:]
          currentBuild.displayName = "#${BUILD_NUMBER}: ${PREFIX}"
        }
        script {
          echo "Clone socok8s git repository"
          sh "git clone --recursive ${git_socok8s_repo} --branch ${git_socok8s_branch} socok8s-git"
          echo "Setup socok8s virtual environment"
          sh """
            set +x
            [ -d ${SOCOK8S_VENV} ] || virtualenv ${SOCOK8S_VENV}
            source ${SOCOK8S_VENV}/bin/activate
            pip install --upgrade -r socok8s-git/script_library/requirements.txt
            python -m ara.setup.env > ${SOCOK8S_VENV}/ara.rc
          """
        }
      }
    }

    stage('Prepare infra') {
      // abort all stages if one of them fails
      failFast true
      parallel {

        stage('Deploy SES') {
          steps {
            script {
              run('deploy_ses')
              get_host_ip('ses_nodes')
            }
          }
        }

        stage('Deploy CaaSP') {
          steps {
            script {
              run('deploy_caasp')
              get_host_ip('caasp-admin')
            }
          }
        }

        stage('Deploy CCP Deployer') {
          steps {
            script {
              run('deploy_ccp_deployer')
              get_host_ip('osh-deployer')
            }
          }
        }
      }
    }

    stage('Enroll CaaSP workers') {
      steps {
        script {
          run('enroll_caasp_workers')
        }
      }
    }

    stage('Prepare CaaSP for OpenStack') {
      steps {
        script {
          run('setup_caasp_workers_for_openstack')
        }
      }
    }

    stage('Deploy OpenStack-Helm') {
      steps {
        script {
          run('deploy_osh')
        }
      }
    }
  }

  post {
    always {
      script {
        hosts.each { host ->
          echo "${host.key} is reacheable at root@${host.value}"
        }
      }
    }
    cleanup {
      script {
        if (cleanup == "always" ||
            cleanup == "on success" && currentBuild.currentResult == "SUCCESS" ||
            cleanup == "on failure" && currentBuild.currentResult != "SUCCESS") {
          echo "Running clean up"
          run('teardown')
        }
      }
      cleanWs()
    }
  }
}

def run(step) {
  sh("""
    set +x
    source ${SOCOK8S_VENV}/bin/activate
    socok8s-git/run.sh "${step}"
  """)
}

def get_host_ip(host) {
  def ip = sh (
    returnStdout: true,
    script: """
      sed -n '/${host}/{n;n;n;p}' ${ANSIBLE_RUNNER_DIR}/inventory/hosts.yml | cut -d':' -f 2
    """
  ).trim()
  hosts.put(host, ip)
  echo """
******************************************************************************
** The ${host} host is reachable at:
**
**        ssh root@${ip}
**
******************************************************************************
  """
}
