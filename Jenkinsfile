// Jenkinsfile
// This pipeline automates the deployment of a PHP application using Jenkins, Ansible, and Docker.
// It consists of three main stages:
// 1. Install Puppet Agent on the Slave node.
// 2. Install Docker on the Slave node using Ansible playbook executed from the Master.
// 3. Build and deploy the PHP Docker container on the Slave node.
// It also includes error handling to clean up the container if the deployment fails.

pipeline {
    // Define the default agent where stages will run.
    // The 'Install Puppet Agent' stage will run on the 'slave' node.
    agent { label 'slave' }

    stages {
        // Stage 1: Install and configure Puppet agent on the slave node (Job 1)
        // This stage runs directly on the 'slave' node.
        stage('Install Puppet Agent') {
            steps {
                script {
                    echo "Updating packages and installing Puppet on current Slave node."
                    sh "sudo apt-get update && sudo apt-get install -y puppet"
                }
            }
        }

        // Stage 2: Push an Ansible configuration to the test server to install Docker (Job 2)
        // This stage MUST run on the Jenkins Master (built-in node) because Ansible is installed there
        // and it needs access to the Ansible project files (hosts, playbook).
        stage('Run Ansible to Install Docker') {
            agent { label 'built-in' } // This stage runs on the Master
            steps {
                script {
                    def masterProjectDir = WORKSPACE // Points to the job's workspace on the Master

                    echo "Ensuring Ansible is installed on Master and running playbook to install Docker on Slave."

                    // Ensure Ansible is installed on the Master. '|| true' prevents pipeline failure if already installed.
                    sh "sudo apt-get update && sudo apt-get install -y ansible || true"

                    // Change directory to the project workspace on the Master to find 'hosts' and 'install-docker.yaml'
                    sh "cd ${masterProjectDir} && ansible-playbook -i hosts install-docker.yaml"
                }
            }
        }

        // Stage 3: Pull the PHP website and Dockerfile, then build and deploy the PHP Docker container (Job 3)
        // This stage MUST also run on the Jenkins Master (built-in node)
        // because it needs to execute SSH commands *from* the Master, using the SSH key located there.
        stage('Build and Deploy Docker Container') {
            agent { label 'built-in' } // CRITICAL FIX: This stage now runs on the Master

            steps {
                script {
                    def masterProjectDir = WORKSPACE // This is the workspace on the Master
                    def sshKeyPath = '/var/lib/jenkins/.ssh/IrelandKey.pem' // Path to SSH key for jenkins user on Master
                    def remoteProjectDir = '/home/ubuntu/php-docker-project' // Desired project directory on the Slave
                    def githubRepo = 'https://github.com/OlanrewajuEniola/php-docker-project.git' // Your GitHub repository URL

                    // Read the slave IP directly from the hosts file.
                    // This is more robust and avoids complex Groovy API calls that cause serialization issues.
                    // Assumes the hosts file is simple and the IP is on the second line.
                    def slaveIp = sh(returnStdout: true, script: "grep 'ansible_user' ${masterProjectDir}/hosts | awk '{print \$1}'").trim()
                    echo "Dynamically determined Slave IP from hosts file: ${slaveIp}"

                    // Fallback in case dynamic lookup fails (though it should work now with grep/awk)
                    if (slaveIp == null || slaveIp.isEmpty()) {
                        echo "Warning: Could not dynamically determine slave IP from hosts file. Using hardcoded fallback."
                        slaveIp = '54.216.198.219' // <--- UPDATE THIS LINE with your latest Slave IP
                    }

                    echo "Cloning or updating project on Slave: ${slaveIp}"
                    // Execute SSH commands from the Master, targeting the Slave
                    sh "ssh -i ${sshKeyPath} ubuntu@${slaveIp} \"git clone ${githubRepo} ${remoteProjectDir} || (cd ${remoteProjectDir} && git pull)\""

                    echo "Building Docker image on Slave: ${slaveIp}"
                    sh "ssh -i ${sshKeyPath} ubuntu@${slaveIp} \"cd ${remoteProjectDir} && docker build --no-cache -t my-php-app .\""

                    echo "Stopping and removing existing container on Slave: ${slaveIp}"
                    sh "ssh -i ${sshKeyPath} ubuntu@${slaveIp} \"docker rm -f php-app || true\""

                    echo "Running new Docker container on Slave: ${slaveIp}"
                    sh "ssh -i ${sshKeyPath} ubuntu@${slaveIp} \"docker run -d -p 80:80 --name php-app my-php-app\""
                }
            }
            // Post-build actions: Define what happens after this stage, especially on failure.
            post {
                failure {
                    script {
                        echo 'Job 3 (Build and Deploy) failed. Attempting to delete the running container on Test Server.'
                        def masterProjectDir = WORKSPACE
                        def sshKeyPath = '/var/lib/jenkins/.ssh/IrelandKey.pem'
                        // Re-read slave IP for cleanup
                        def slaveIp = sh(returnStdout: true, script: "grep 'ansible_user' ${masterProjectDir}/hosts | awk '{print \$1}'").trim()
                        if (slaveIp == null || slaveIp.isEmpty()) {
                            slaveIp = '54.216.198.219' // Fallback
                        }
                        sh "ssh -i ${sshKeyPath} ubuntu@${slaveIp} \"docker rm -f php-app || true\""
                        echo 'Container deletion attempt completed.'
                    }
                }
                success {
                    echo 'PHP application built and deployed successfully on the Test Server!'
                }
                always {
                    echo 'Finished Build and Deploy Docker Container stage.'
                }
            }
        }
    }
}
