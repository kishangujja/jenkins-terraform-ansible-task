pipeline {
    agent any

    environment {
        AWS_ACCESS_KEY_ID = credentials('aws')  // ID from Jenkins credentials store
        AWS_SECRET_ACCESS_KEY = credentials('aws')  // ID from Jenkins credentials store
        SH_KEY_NAME = "jenkins_rsa"
        SSH_KEY_PATH = "${env.WORKSPACE}/.ssh/${SH_KEY_NAME}"
        SSH_PUBLIC_KEY_PATH = "${env.WORKSPACE}/.ssh/${SH_KEY_NAME}.pub"
        INVENTORY_FILE = "${env.WORKSPACE}/inventory.yaml"  // Adjust path based on actual location
        WORKER_GROUP = "frontend"  // Set to either 'frontend' or 'backend', adjust as needed
        
    }

    stages {

        stage('Checkout') {
            steps {
                deleteDir()  // Clean workspace
                echo 'Cloning repository...'
                sh 'git clone https://github.com/kishangujja/jenkins-terraform-ansible-task.git'
                sh 'ls -al'  // List files to check if inventory.yaml is there
            }
        }

        stage('Terraform Apply') {
            steps {
                script {
                    dir("${env.WORKSPACE}/jenkins-terraform-ansible-task") {
                        sh 'pwd'  // Print current working directory
                        sh 'terraform init'
                        sh 'terraform validate'
                        sh 'terraform plan'
                        sh 'terraform apply -auto-approve'
                    }
                }
            }
        }

        stage('Generate SSH Key Pair') {
            steps {
                script {
                    // Generate SSH key pair if it doesn't already exist
                    if (!fileExists(SSH_KEY_PATH)) {
                        sh """
                            mkdir -p ${env.WORKSPACE}/.ssh
                            ssh-keygen -t ed25519 -f ${SSH_KEY_PATH} -N '' -C 'jenkins@controller' || true
                        """
                    }
                }
            }
        }

        stage('Load Inventory and Get Worker Node') {
            steps {
                script {
                    echo "Inventory file path: ${INVENTORY_FILE}"  // Debugging: Check the file path
                    def inventory = readYaml file: "${INVENTORY_FILE}"
                    
                    // Debugging: Print the loaded inventory content
                    echo "Loaded inventory: ${inventory}"
                    
                    // Load the correct worker group (frontend or backend)
                    def workerGroup = inventory[WORKER_GROUP]
                    if (!workerGroup) {
                        error "Worker group '${WORKER_GROUP}' not found in the inventory"
                    }

                    // Pick the first worker node in the group
                    def workerNode = workerGroup[0]

                    // Extract worker node details
                    WORKER_NODE_IP = workerNode.ip
                    WORKER_NODE_USER = workerNode.user

                    echo "Using worker node IP: ${WORKER_NODE_IP}, User: ${WORKER_NODE_USER}"
                }
            }
        }

        stage('Ensure Passwordless SSH Login to Worker Node') {
            steps {
                script {
                    // Ensure the worker node allows passwordless SSH login
                    sh """
                        # First, ensure that SSH password authentication is enabled on the worker node
                        ssh -o StrictHostKeyChecking=no ${WORKER_NODE_USER}@${WORKER_NODE_IP} 'echo "PermitEmptyPasswords yes" | sudo tee -a /etc/ssh/sshd_config'
                        ssh -o StrictHostKeyChecking=no ${WORKER_NODE_USER}@${WORKER_NODE_IP} 'sudo systemctl restart sshd'

                        # Copy the public key to the worker node if not already there
                        ssh-copy-id -i ${SSH_PUBLIC_KEY_PATH} ${WORKER_NODE_USER}@${WORKER_NODE_IP} || true

                        # Ensure that the permissions for the .ssh folder and authorized_keys are correct
                        ssh -o StrictHostKeyChecking=no ${WORKER_NODE_USER}@${WORKER_NODE_IP} 'mkdir -p ~/.ssh && chmod 700 ~/.ssh'
                        ssh -o StrictHostKeyChecking=no ${WORKER_NODE_USER}@${WORKER_NODE_IP} 'chmod 600 ~/.ssh/authorized_keys'
                    """
                }
            }
        }

        stage('Test SSH Connection') {
            steps {
                script {
                    // Test the SSH connection from Jenkins to the worker node using the private key
                    sshagent (credentials: ['jenkins-ssh-key']) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ${WORKER_NODE_USER}@${WORKER_NODE_IP} "echo 'SSH Connection Successful!'"
                        """
                    }
                }
            }
        }

        stage('Build on Worker Node') {
            agent { label 'worker-node' }  // Replace with your worker node label if necessary
            steps {
                echo "Running the build on the worker node..."
                // Add your build steps here, for example:
                sh 'echo "Building on worker node"'
            }
        }

        stage('Ansible Deployment') {
            steps {
                script {
                   sleep '30'
                   ansiblePlaybook becomeUser: 'ec2-user', credentialsId: 'aws', disableHostKeyChecking: true, installation: 'ansible', inventory: '/var/lib/jenkins/workspace/ansible-terraform-jenkins/jenkins-terraform-ansible-task/inventory.yaml', playbook: '/var/lib/jenkins/workspace/ansible-terraform-jenkins/jenkins-terraform-ansible-task/amazon-playbook.yml', vaultTmpPath: ''
                   ansiblePlaybook become: true, credentialsId: 'aws', disableHostKeyChecking: true, installation: 'ansible', inventory: '/var/lib/jenkins/workspace/ansible-terraform-jenkins/jenkins-terraform-ansible-task/inventory.yaml', playbook: '/var/lib/jenkins/workspace/ansible-terraform-jenkins/jenkins-terraform-ansible-task/ubuntu-playbook.yml', vaultTmpPath: ''
                }
            }
        }
    }
}
