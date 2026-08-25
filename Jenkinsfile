pipeline {
  agent any

  parameters {

    choice(
      name: 'TARGET',
      choices: [
        'linux_servers',
        "server01"
        ],
      description: 'Select Linux Servers/group to patch'
      )
  }

  environment {
        ANSIBLE_HOST = '10.0.10.10'
        ANSIBLE_USER = 'itadmin'
        REMOTE_DIR = '/data/linux-patching_jenkinspipeline1'
    }

  stages {

        // ==============================================
        // 1. GITHUB CHECKOUT
        // ==============================================

        stage('Git Checkout') {

            steps {

                checkout scm

                sh '''
                    echo "======================================"
                    echo "Git checkout completed"
                    echo "======================================"

                    git log -1 --oneline

                    echo ""
                    echo "Files:"
                    find . -maxdepth 3 -type f
                '''
            }
        }
  
        // ==============================================
        // 2. COPY CODE TO ANSIBLE VM
        // ==============================================

        stage('Copy Playbook to Ansible Server') {

           steps {

              sshagent(credentials: ['esrjenkins-esrittool-ansible-ssh']) {

                sh """
                  ssh \
                       -o StrictHostKeyChecking=no \
                         ${ANSIBLE_USER}@${ANSIBLE_HOST} \
                         "mkdir -p ${REMOTE_DIR}"

                   scp \
                        -o StrictHostKeyChecking=no \
                        -r ./ansible ${ANSIBLE_USER}@${ANSIBLE_HOST}:${REMOTE_DIR}/
                    """
                }
            }
        }


         // ==============================================
        // 3. ANSIBLE SYNTAX CHECK FOR PRE PATCH
        // ==============================================

        stage('Ansible Syntax Check') {

            steps {

                sshagent(credentials: ['esrjenkins-esrittool-ansible-ssh']) {

                    sh """

                        ssh \
                          -o StrictHostKeyChecking=no \
                          ${ANSIBLE_USER}@${ANSIBLE_HOST} \
                          "cd ${REMOTE_DIR} && \
                           ansible-playbook \
                           -i ansible/inventory/production.ini \
                           ansible/playbooks/pre_patch.yml \
                           --syntax-check"
                    """
                }
            }
        
        // ==============================================
        // 3. ANSIBLE SYNTAX CHECK FOR PATCH
        // ==============================================

           steps {

                sshagent(credentials: ['esrjenkins-esrittool-ansible-ssh']) {

                    sh """

                        ssh \
                          -o StrictHostKeyChecking=no \
                          ${ANSIBLE_USER}@${ANSIBLE_HOST} \
                          "cd ${REMOTE_DIR} && \
                           ansible-playbook \
                           -i ansible/inventory/production.ini \
                           ansible/playbooks/patch.yml \
                           --syntax-check"
                    """
                }
            }
        }
        // ==============================================
        // 4. PRE-CHECK
        // ==============================================

        stage('Pre-Check') {

            steps {

                sshagent(credentials: ['esrjenkins-esrittool-ansible-ssh']) {

                    sh """

                        ssh \
                          -o StrictHostKeyChecking=no \
                          ${ANSIBLE_USER}@${ANSIBLE_HOST} \
                          "cd ${REMOTE_DIR} && \
                           ansible \
                           -i ansible/inventory/production.ini \
                           ${TARGET} \
                           -m ping"
                    """
                }
            }

            steps {
                sshagent(credentials: ['esrjenkins-esrittool-ansible-ssh']) {

                    sh """

                        ssh \
                          -o StrictHostKeyChecking=no \
                          ${ANSIBLE_USER}@${ANSIBLE_HOST} \
                          "cd ${REMOTE_DIR} && \
                           ansible-playbook \
                           -i ansible/inventory/production.ini \
                           ansible/playbooks/pre_patch.yml \
                           --limit ${TARGET}"
                           
                    """
                }
            }   
        }

        // ==============================================
        // 5. APPROVAL
        // ==============================================

        stage('Approval') {

            when {
                expression {
                    params.PRODUCTION_APPROVAL
                }
            }

            steps {

                input(
                    message: "Proceed with Linux patching for ${TARGET}?",
                    ok: "Patch Linux Servers"
                )
            }
        }

        // ==============================================
        // 6. PATCH
        // ==============================================

        stage('Linux Patching') {

            steps {

                sshagent(credentials: ['esrjenkins-esrittool-ansible-ssh']) {

                    sh """

                        ssh \
                          -o StrictHostKeyChecking=no \
                          ${ANSIBLE_USER}@${ANSIBLE_HOST} \
                          "cd ${REMOTE_DIR} && \
                           ansible-playbook \
                           -i ansible/inventory/production.ini \
                           ansible/playbooks/patch.yml \
                           --limit ${TARGET}"
                    """
                }
            }
        }
    
    post {

        success {

            echo '''
            ==========================================
            LINUX PATCHING SUCCESSFUL
            ==========================================
            '''
        }

        failure {

            echo '''
            ==========================================
            LINUX PATCHING FAILED
            ==========================================
            '''
        }

        always {

            echo "Target: ${TARGET}"
            echo "Ansible VM: ${ANSIBLE_HOST}"
        }
    }
}
