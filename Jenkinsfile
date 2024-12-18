pipeline {
    agent any

    environment {
        AWS_ACCESS_KEY_ID = credentials('aws')  // ID from Jenkins credentials store
        AWS_SECRET_ACCESS_KEY = credentials('aws')  // ID from Jenkins credentials store
        SH_KEY_NAME = "jenkins_rsa"
        SSH_KEY_PATH = "${env.WORKSPACE}/.ssh/${SH_KEY_NAME}"
        SSH_PUBLIC_KEY_PATH = "${env.WORKSPACE}/.ssh/${SH_KEY_NAME}.pub"
        INVENTORY_FILE = "${env.WORKSPACE}/inventory.yaml"  // Corrected path to inventory.yaml
        
    }

    stages {

        stage('Checkout') {
            steps {
                deleteDir()
                echo 'Cloning repository...'
                sh 'git clone https://github.com/kishangujja/jenkins-terraform-ansible-task.git'
            }
        }

        stage('Terraform Apply') {
            steps {
                script {
                    dir("${env.WORKSPACE}/jenkins-terraform-ansible-task") {
                        sh 'pwd'
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
                    // Debugging: Read file content and echo it
                    def yamlContent = readFile("${INVENTORY_FILE}")
                    echo "YAML Content: ${yamlContent}"  // Logs the content of the YAML file

                    // Load YAML content into a Groovy object
                    def inventory = readYaml file: "${INVENTORY_FILE}"
                    
                    // Check if the group exists in the inventory
                    def workerGroup = inventory[WORKER_GROUP]
                    if (!workerGroup) {
                        error "Worker group '${WORKER_GROUP}' not found in the inventory"
                    }

                    // Pick the first worker node in the group
                    def workerNode = workerGroup[0]  // Use the first worker node in the selected group

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
                    sh """
                        # Ensure passwordless SSH access to worker node
                        ssh -o StrictHostKeyChecking=no ${WORKER_NODE_USER}@${WORKER_NODE_IP} 'mkdir -p ~/.ssh && chmod 700 ~/.ssh'
                        ssh -o StrictHostKeyChecking=no ${WORKER_NODE_USER}@${WORKER_NODE_IP} 'echo ${WORKER_NODE_PUBLIC_KEY} >> ~/.ssh/authorized_keys'
                        ssh -o StrictHostKeyChecking=no ${WORKER_NODE_USER}@${WORKER_NODE_IP} 'chmod 600 ~/.ssh/authorized_keys'
                    """
                }
            }
        }

        stage('Test SSH Connection') {
            steps {
                script {
                    sshagent (credentials: ['jenkins-ssh-key']) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ${WORKER_NODE_USER}@${WORKER_NODE_IP} "echo 'SSH Connection Successful!'"
                        """
                    }
                }
            }
        }

        stage('Build on Worker Node') {
            agent { label 'worker-node' }
            steps {
                echo "Running the build on the worker node..."
                sh 'echo "Building on worker node"'
            }
        }

        stage('Ansible Deployment') {
            steps {
                script {
                    // First Ansible playbook deployment (Amazon-based)
                    ansiblePlaybook(
                        becomeUser: 'ec2-user',
                        credentialsId: 'aws',
                        disableHostKeyChecking: true,
                        installation: 'ansible',
                        inventory: "${INVENTORY_FILE}",
                        playbook: "${env.WORKSPACE}/amazon-playbook.yml"  // Corrected path to amazon-playbook.yml
                    )

                    // Second Ansible playbook deployment (Ubuntu-based)
                    ansiblePlaybook(
                        become: true,
                        credentialsId: 'aws',
                        disableHostKeyChecking: true,
                        installation: 'ansible',
                        inventory: "${INVENTORY_FILE}",
                        playbook: "${env.WORKSPACE}/ubuntu-playbook.yml"  // Corrected path to ubuntu-playbook.yml
                    )
                }
            }
        }
    }
}
