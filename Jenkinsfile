pipeline {
    agent { label 'slave' }
    options {
        cleanWs() // This ensures a clean workspace for each build.
                  // It prevents leftover 'hosts' files (and other artifacts) from previous runs causing conflicts.
    }

    stages {
        // Jenkins's Declarative Pipeline implicitly handles the SCM checkout at the start.
        // The repository content will be in the default workspace: /home/ubuntu/workspace/php-docker-pipeline

        stage('Install Puppet Agent') {
            steps {
                sh 'sudo apt-get update && sudo apt-get install -y puppet'
            }
        }

        stage('Prepare Ansible Hosts') {
            steps {
                // Write the 'hosts' file directly into the Jenkins workspace (which is the current directory for this stage).
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
        failure {
            sh 'docker rm -f php-app || true'
        }
    }
}
