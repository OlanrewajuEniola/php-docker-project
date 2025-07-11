pipeline {
    agent { label 'slave' }

    stages {

	stage('Clone or Pull GitHub Repo') {
            steps {
                sh '''
                   if [ ! -d /home/ubuntu/php-docker-project ]; then
                       git clone https://github.com/OlanrewajuEniola/php-docker-project.git /home/ubuntu/php-docker-project
                   else
                       cd /home/ubuntu/php-docker-project && git pull origin main
                   fi
                '''
            }
        }

        stage('Install Puppet Agent') {
            steps {
                sh 'sudo apt-get update && sudo apt-get install -y puppet'
            }
        }

 	stage('Prepare Ansible Hosts') {
            steps {
                sh '''
                   echo "[slaves]" > /home/ubuntu/php-docker-project/hosts
                   echo "34.253.105.138 ansible_user=ubuntu" >> /home/ubuntu/php-docker-project/hosts
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
        	sh 'sudo apt-get update && sudo apt-get install -y ansible'
        	sh 'cd /home/ubuntu/php-docker-project && ansible-playbook -i hosts install-docker.yaml'
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
