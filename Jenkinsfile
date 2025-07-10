pipeline {
    agent { label 'slave' }

    stages {
        stage('Install Puppet Agent') {
            steps {
                sh 'sudo apt-get update && sudo apt-get install -y puppet'
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
                sh 'sudo apt-get update && sudo apt-get install -y ansible && ansible-playbook -i hosts install-docker.yaml'
            }
        }

        stage('Build and Deploy Docker Container') {
            steps {

 		// Clone or pull latest from GitHub on slave node
    		sh 'cd /home/ubuntu/php-docker-project || git clone https://github.com/OlanrewajuEniola/php-docker-project.git /home/ubuntu/php-docker-project'
    		sh 'cd /home/ubuntu/php-docker-project && git pull origin main'

 		// Build the Docker image
    		sh 'cd /home/ubuntu/php-docker-project && docker build -t my-php-app .'

    		// Remove any running container named php-app (ignore errors)
    		sh 'docker rm -f php-app || true'

    		// Run the container mapping port 80
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
