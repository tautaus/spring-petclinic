pipeline {
    agent any

    environment {
        SONARQUBE_URL = 'http://sonarqube:9000'
        SONARQUBE_CREDENTIALS_ID = 'admin'
        GITHUB_TOKEN = $GITHUB_TOKEN
    }

    stages {
        // stage('Checkout') {
        //     steps {
        //         script {
        //             echo "Checking out code..."
        //             git url: 'https://github.com/CChariot/spring-petclinic.git', branch: 'FinalProject_main', credentialsId: 'github-token'
        //         }
        //     }
        // }

        // stage('Build JAR') {
        //     steps {
        //         script {
        //             echo "Building JAR..."
        //             sh './mvnw clean package -Dmaven.test.skip=true'
        //         }
        //     }
        // }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker Image..."
                    def dockerImage = docker.build("spring-petclinic", "--no-cache .")
                    echo "Docker Image built: ${dockerImage.id}"
                    // Store the Docker image ID in the environment if needed across stages
                    env.DOCKER_IMAGE_ID = dockerImage.id
                }
            }
        }

        stage('Deploy Container to VM') {
            steps {
                ansiblePlaybook(
                    playbook: 'deploy.yml',
                    inventory: 'inventory'
                )
            }
        }

        // Further stages would reference env.DOCKER_IMAGE_ID if needed
    }

    post {
        always {
            script {
                // Use the saved Docker image ID from the environment if needed
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
