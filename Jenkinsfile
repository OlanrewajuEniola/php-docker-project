// Jenkinsfile
// This pipeline automates the deployment of a PHP application using Jenkins, Ansible, and Docker.
// It consists of three main stages:
// 1. Install Puppet Agent on the Slave node.
// 2. Install Docker on the Slave node using Ansible playbook executed from the Master.
// 3. Build and deploy the PHP Docker container on the Slave node.
// It also includes error handling to clean up the container if the deployment fails.

pipeline {
    // Define the default agent where stages will run.
    // Most stages (Puppet, Docker Build/Deploy) will run on the 'slave' node.
    agent { label 'slave' }

    stages {
        // Stage 1: Install and configure Puppet agent on the slave node (Job 1)
        // This stage runs directly on the 'slave' node, so no SSH is needed to reach it.
        stage('Install Puppet Agent') {
            steps {
                script {
                    echo "Updating packages and installing Puppet on current Slave node."
                    // Execute commands directly on the slave agent
                    sh "sudo apt-get update && sudo apt-get install -y puppet"
                }
            }
        }

        // Stage 2: Push an Ansible configuration to the test server to install Docker (Job 2)
        // This stage needs to run on the Jenkins Master (built-in node) because Ansible is installed there
        // and it needs access to the Ansible project files (hosts, playbook).
        stage('Run Ansible to Install Docker') {
            agent { label 'built-in' } // CRITICAL FIX: This stage will run on the Master (built-in) node

            steps {
                script {
                    // Clean the workspace before checkout to ensure no old files interfere
                    cleanWs() // Added to ensure a clean workspace
                    // Explicitly checkout the SCM again for this stage to ensure latest files are used
                    checkout scm // Added to ensure the latest hosts file is pulled

                    // WORKSPACE is a built-in Jenkins environment variable that points to the
                    // current job's workspace directory on the agent where this stage is running (Master).
                    def masterProjectDir = WORKSPACE // CRITICAL FIX: Use WORKSPACE for the correct path

                    echo "Ensuring Ansible is installed on Master and running playbook to install Docker on Slave."

                    // Ensure Ansible is installed on the Master. '|| true' prevents pipeline failure if already installed.
                    sh "sudo apt-get update && sudo apt-get install -y ansible || true"

                    // Change directory to the project workspace on the Master to find 'hosts' and 'install-docker.yaml'
                    // Then execute the Ansible playbook targeting the slave.
                    sh "cd ${masterProjectDir} && ansible-playbook -i hosts install-docker.yaml"
                }
            }
        }

        // Stage 3: Pull the PHP website and Dockerfile, then build and deploy the PHP Docker container (Job 3)
        // This stage runs on the 'slave' node, but it uses SSH to execute Docker commands on the slave itself.
        stage('Build and Deploy Docker Container') {
            // This stage will use the pipeline's default agent: 'slave'
            steps {
                script {
                    // Dynamically get the IP address of the 'slave' node from Jenkins's internal information.
                    // This is a more robust way to get the IP address.
                    // NOTE: This line might require 'In-process Script Approval' in Jenkins if it's the first time running it.
                    def slaveNode = Jenkins.instance.getNode('slave')
                    def slaveIp = ''
                    if (slaveNode != null) {
                        // Attempt to get the IP from the node's properties, which is more reliable
                        def sshLauncher = slaveNode.getLauncher()
                        if (sshLauncher instanceof hudson.plugins.sshslaves.SSHLauncher) {
                            slaveIp = sshLauncher.getHost()
                        } else {
                            // Fallback if not SSHLauncher or if getHost() doesn't work
                            slaveIp = slaveNode.getComputer().getEnvironment().get('HOSTNAME') // Try hostname
                            if (slaveIp == null || slaveIp.isEmpty() || slaveIp.contains('localhost')) {
                                slaveIp = slaveNode.getComputer().getName() // Try node name
                            }
                        }
                    }

                    if (slaveIp == null || slaveIp == '') { // Changed .isEmpty() to == '' for robustness
                        echo "Warning: Could not dynamically determine slave IP. Using hardcoded fallback."
                        slaveIp = '54.195.215.77' // <--- UPDATE THIS LINE with your latest Slave IP
                    }

                    def sshKeyPath = '/var/lib/jenkins/.ssh/IrelandKey.pem' // Path to your SSH key for jenkins user on the Master
                    def remoteProjectDir = '/home/ubuntu/php-docker-project' // Desired project directory on the Slave
                    def githubRepo = 'https://github.com/OlanrewajuEniola/php-docker-project.git' // Your GitHub repository URL

                    echo "Cloning or updating project on Slave: ${slaveIp}"
                    // Use SSH to clone the repo on the slave. If it already exists, pull latest changes.
                    sh "ssh -i ${sshKeyPath} ubuntu@${slaveIp} \"git clone ${githubRepo} ${remoteProjectDir} || (cd ${remoteProjectDir} && git pull)\""

                    echo "Building Docker image on Slave: ${slaveIp}"
                    // Use SSH to navigate to the project directory on the slave and build the Docker image.
                    sh "ssh -i ${sshKeyPath} ubuntu@${slaveIp} \"cd ${remoteProjectDir} && docker build --no-cache -t my-php-app .\""

                    echo "Stopping and removing existing container on Slave: ${slaveIp}"
                    // Use SSH to stop and remove any existing container named 'php-app'. '|| true' prevents failure if not found.
                    sh "ssh -i ${sshKeyPath} ubuntu@${slaveIp} \"docker rm -f php-app || true\""

                    echo "Running new Docker container on Slave: ${slaveIp}"
                    // Use SSH to run the new Docker container in detached mode, mapping port 80.
                    sh "ssh -i ${sshKeyPath} ubuntu@${slaveIp} \"docker run -d -p 80:80 --name php-app my-php-app\""
                }
            }
            // Post-build actions: Define what happens after this stage, especially on failure.
            post {
                failure {
                    script {
                        echo 'Job 3 (Build and Deploy) failed. Attempting to delete the running container on Test Server.'
                        // Re-determine slave IP for post-failure cleanup, or use fallback
                        def slaveNode = Jenkins.instance.getNode('slave')
                        def slaveIp = ''
                        if (slaveNode != null) {
                            def sshLauncher = slaveNode.getLauncher()
                            if (sshLauncher instanceof hudson.plugins.sshslaves.SSHLauncher) {
                                slaveIp = sshLauncher.getHost()
                            } else {
                                slaveIp = slaveNode.getComputer().getEnvironment().get('HOSTNAME')
                                if (slaveIp == null || slaveIp.isEmpty() || slaveIp.contains('localhost')) {
                                    slaveIp = slaveNode.getComputer().getName()
                                }
                            }
                        }
                        if (slaveIp == null || slaveIp == '') { // Changed .isEmpty() to == '' for robustness
                            slaveIp = '54.195.215.77' // <--- UPDATE THIS LINE with your latest Slave IP
                        }
                        def sshKeyPath = '/var/lib/jenkins/.ssh/IrelandKey.pem' // Path to your SSH key on the Master
                        // Forcefully remove the container on the slave. '|| true' ensures the cleanup itself doesn't fail.
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

