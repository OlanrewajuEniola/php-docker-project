pipeline {
    agent { label 'slave' }

    stages {
        stage('Install Puppet Agent') {
            steps {
                sh 'sudo apt-get update && sudo apt-get install -y puppet'
            }
        }

        stage('Run Ansible to Install Docker') {
            steps {
                sh 'sudo apt-get update && ansible-playbook -i hosts install-docker.yml'
            }
        }

        stage('Build and Deploy Docker Container') {
            steps {
                sh 'cd /home/ubuntu/php-docker-project && docker build -t my-php-app .'
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
