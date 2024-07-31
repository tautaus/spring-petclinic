pipeline {
    agent any

    environment {
        SONARQUBE_URL = "http://sonarqube:9000"
        SONARQUBE_CREDENTIALS_ID = "admin"
        GITHUB_TOKEN = "${GITHUB_TOKEN}"
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    echo "Checking out code..."
                    git url: 'https://github.com/CChariot/spring-petclinic.git', branch: 'FinalProject_main', credentialsId: 'github-token'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker Image..."
                    def dockerImage = docker.build("spring-petclinic", "--no-cache .")
                    echo "Docker Image built: ${dockerImage.id}"
                    env.DOCKER_IMAGE_ID = dockerImage.id
                }
            }
        }

        stage('Add SSH Key and Deploy Container to VM') {
            steps {
                script {
                    // Add the SSH key to known hosts
                    sh '''
                        mkdir -p ~/.ssh
                        chmod 700 ~/.ssh
                        ssh-keyscan -H 13.58.183.76 >> ~/.ssh/known_hosts
                        chmod 600 ~/.ssh/known_hosts
                    '''
                    // Execute the Ansible playbook
                    ansiblePlaybook(
                        playbook: '/var/jenkins_home/deploy.yml',
                        inventory: '/var/jenkins_home/inventory',
                        extras: '--private-key=/var/jenkins_home/ansible.pem'
                    )
                }
            }
        }
    }

    post {
        always {
            script {
                if (env.DOCKER_IMAGE_ID) {
                    echo "Stopping and removing Docker Image with ID: ${env.DOCKER_IMAGE_ID}"
                    sh "docker rmi -f ${env.DOCKER_IMAGE_ID}"
                }
            }
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}
