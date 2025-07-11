pipeline {
    agent { label 'slave' }
    options {
        // 1. Add cleanWs() option to ensure a fresh workspace for each build.
        // This prevents leftover 'hosts' files (and other artifacts) from causing conflicts.
        cleanWs()
    }

    stages {
        // 'Declarative: Checkout SCM' implicitly handles cloning the repo into
        // /home/ubuntu/workspace/php-docker-pipeline on your Slave1.

        stage('Install Puppet Agent') {
            steps {
                sh 'sudo apt-get update && sudo apt-get install -y puppet'
            }
        }

        stage('Prepare Ansible Hosts') {
            steps {
                // 2. Correct the path for 'hosts' file creation.
                // It should be relative to the workspace, not an absolute path.
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
                // 3. Add -vvv for verbose Ansible output (helpful for debugging).
                // No 'cd' is needed if 'hosts' and 'install-docker.yaml' are in the workspace root.
                sh '''
                    sudo apt-get update
                    sudo apt-get install -y ansible
                    ansible-playbook -i hosts install-docker.yaml -vvv
                '''
            }
        }

        stage('Build and Deploy Docker Container') {
            steps {
                // No 'cd' is needed if 'Dockerfile' is in the workspace root.
                sh 'docker build -t my-php-app .'
                sh 'docker rm -f php-app || true'
                sh 'docker run -d -p 80:80 --name php-app my-php-app'
            }
        }
    }

    post {
        failure {
            sh 'docker rm -f php-app || true'
        }
    }
}
