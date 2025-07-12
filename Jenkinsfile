// Jenkinsfile
pipeline {
    agent { label 'slave' } // This specifies that most stages will run on the agent labeled 'slave'

    stages {
        stage('Install Puppet Agent') { // This is Job 1
            steps {
                script {
                    def slaveIp = '18.203.232.61' // Your Slave node's Public IP
                    def sshKeyPath = '/home/ubuntu/.ssh/IrelandKey.pem' // Path to your SSH key on the Master

                    echo "Updating packages and installing Puppet on Slave: ${slaveIp}"
                    sh "ssh -i ${sshKeyPath} ubuntu@${slaveIp} \"sudo apt-get update && sudo apt-get install -y puppet\""
                }
            }
        }

        stage('Run Ansible to Install Docker') { // This is Job 2
            agent { label 'built-in' } // <--- IMPORTANT: This stage will run on the Jenkins built-in node (your Master EC2)
            steps {
                script {
                    def masterProjectDir = '/home/ubuntu/php-docker-project' // Project directory on the Master where ansible files are

                    echo "Ensuring Ansible is installed on Master and running playbook to install Docker on Slave."
                    // These commands will now run directly on the Jenkins built-in node (your Master EC2)
                    sh "sudo apt-get update && sudo apt-get install -y ansible || true" // Ensure Ansible is installed on Master
                    sh "cd ${masterProjectDir} && ansible-playbook -i hosts install-docker.yaml"
                }
            }
        }

        stage('Build and Deploy Docker Container') { // This is Job 3
            // This stage will use the pipeline's default agent: 'slave'
            steps {
                script {
                    def slaveIp = '18.203.232.61' // Your Slave node's Public IP
                    def sshKeyPath = '/home/ubuntu/.ssh/IrelandKey.pem' // Path to your SSH key on the Master
                    def remoteProjectDir = '/home/ubuntu/php-docker-project' // Directory on the Slave
                    def githubRepo = 'https://github.com/OlanrejEniola/php-docker-project.git' // Corrected repo URL as per provided snippets

                    echo "Cloning or updating project on Slave: ${slaveIp}"
                    sh "ssh -i ${sshKeyPath} ubuntu@${slaveIp} \"git clone ${githubRepo} ${remoteProjectDir} || (cd ${remoteProjectDir} && git pull)\""

                    echo "Building Docker image on Slave: ${slaveIp}"
                    sh "ssh -i ${sshKeyPath} ubuntu@${slaveIp} \"cd ${remoteProjectDir} && docker build --no-cache -t my-php-app .\""

                    echo "Stopping and removing existing container on Slave: ${slaveIp}"
                    sh "ssh -i ${sshKeyPath} ubuntu@${slaveIp} \"docker rm -f php-app || true\""

                    echo "Running new Docker container on Slave: ${slaveIp}"
                    sh "ssh -i ${sshKeyPath} ubuntu@${slaveIp} \"docker run -d -p 80:80 --name php-app my-php-app\""
                }
            }
            post {
                failure {
                    script {
                        echo 'Job 3 (Build and Deploy) failed. Attempting to delete the running container on Test Server.'
                        def slaveIp = '18.203.232.61' // Ensure this is the correct Slave node IP
                        def sshKeyPath = '/home/ubuntu/.ssh/IrelandKey.pem' // Path to your SSH key on the Master
// Path to your SSH key on the Master
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
