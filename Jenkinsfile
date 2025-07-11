pipeline {
    agent { label 'slave' }

    stages {
        stage('Install Puppet Agent') {
            steps {
                script {
                    echo "Updating packages and installing Puppet on current Slave node."
                    sh "sudo apt-get update && sudo apt-get install -y puppet"
                }
            }
        }

        stage('Run Ansible to Install Docker') {
            agent { label 'built-in' } // CRITICAL FIX: This stage will run on the Master (built-in) node

            steps {
                script {
                    def masterProjectDir = WORKSPACE // CRITICAL FIX: Use WORKSPACE for the correct path

                    echo "Ensuring Ansible is installed on Master and running playbook to install Docker on Slave."

                    sh "sudo apt-get update && sudo apt-get install -y ansible || true"

                    sh "cd ${masterProjectDir} && ansible-playbook -i hosts install-docker.yaml"
                }
            }
        }
        stage('Build and Deploy Docker Container') {
            steps {
                script {
                    def slaveNode = Jenkins.instance.getNode('slave')
                    def slaveIp = ''
                    if (slaveNode != null) {
                        def sshLauncher = slaveNode.getLauncher()
                        if (sshLauncher instanceof hudson.plugins.sshslaves.SSHLauncher) {
                            slaveIp = sshLauncher.getHost()
                        } else {
                            slaveIp = slaveNode.getComputer().getEnvironment().get('HOSTNAME') // Try hostname
                            if (slaveIp == null || slaveIp.isEmpty() || slaveIp.contains('localhost')) {
                                slaveIp = slaveNode.getComputer().getName() // Try node name
                            }
                        }
                    }

                    if (slaveIp == null || slaveIp.isEmpty()) {
                        echo "Warning: Could not dynamically determine slave IP. Using hardcoded fallback."
                        slaveIp = '34.244.40.80' // <--- UPDATE THIS LINE with your latest Slave IP
                    }

                    def sshKeyPath = '/home/ubuntu/.ssh/IrelandKey.pem' // Path to your SSH key on the Master
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
            post {
                failure {
                    script {
                        echo 'Job 3 (Build and Deploy) failed. Attempting to delete the running container on Test Server.'
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
                        if (slaveIp == null || slaveIp.isEmpty()) {
                            slaveIp = '34.244.40.80' // <--- UPDATE THIS LINE with your latest Slave IP
                        }
                        def sshKeyPath = '/home/ubuntu/.ssh/IrelandKey.pem' // Path to your SSH key on the Master
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
