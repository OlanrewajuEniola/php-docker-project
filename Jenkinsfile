pipeline {
    agent { label 'slave' }
    // REMOVE the 'options { cleanWs() }' block entirely from here.
    // cleanWs() is not a valid option type.

    stages {
        stage('Install Puppet Agent') {
            steps {
                sh 'sudo apt-get update && sudo apt-get install -y puppet'
            }
        }

        stage('Prepare Ansible Hosts') {
            steps {
                sh '''
                    echo "[slaves]" > hosts
                    echo "34.253.105.138 ansible_user=ubuntu ansible_ssh_private_key_file=/home/ubuntu/.ssh/IrelandKey.pem" >> hosts
                '''
            }
        }

        stage('Debug Ansible') {
            steps {
                sh 'which ansible-playbook'
                sh 'ansible-playbook --version'
            }
        }

        stage('Run Ansible to Install Docker') {
            steps {
                sh '''
                    sudo apt-get update
                    sudo apt-get install -y ansible
                    ansible-playbook -i hosts install-docker.yaml -vvv
                '''
            }
        }

        stage('Build and Deploy Docker Container') {
            steps {
                sh 'docker build -t my-php-app .'
                sh 'docker rm -f php-app || true'
                sh 'docker run -d -p 80:80 --name php-app my-php-app'
            }
        }
    }

    post {
        always { // This ensures cleanWs() runs whether the build succeeds or fails
            cleanWs() // <-- Place cleanWs() here as a post-build step
        }
        failure {
            sh 'docker rm -f php-app || true'
        }
    }
}
