stage('Build and Deploy Docker Container') { // This is Job 3
            // This stage will use the pipeline's default agent: 'slave'
            steps {
                script {
                    // Dynamically get the IP address of the 'slave' node
                    def slaveNode = Jenkins.instance.getNode('slave')
                    def slaveIp = slaveNode.getComputer().getDescriptor().getIpAddress(slaveNode)
                    if (slaveIp == null) {
                        // Fallback or error if IP cannot be determined, or use a known static IP if preferred
                        // For a quick fix, you might still hardcode it here if dynamic lookup fails
                        echo "Warning: Could not dynamically determine slave IP. Using hardcoded fallback."
                        slaveIp = '18.203.232.61' // Fallback to your last known IP
                    }

                    def sshKeyPath = '/home/ubuntu/.ssh/IrelandKey.pem' // Path to your SSH key on the Master
                    def remoteProjectDir = '/home/ubuntu/php-docker-project' // Directory on the Slave
                    def githubRepo = 'https://github.com/OlanrewajuEniola/php-docker-project.git'

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
                        // Use the dynamically determined slaveIp for post-build actions as well
                        def slaveNode = Jenkins.instance.getNode('slave')
                        def slaveIp = slaveNode.getComputer().getDescriptor().getIpAddress(slaveNode)
                        if (slaveIp == null) {
                            slaveIp = '18.203.232.61' // Fallback
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
