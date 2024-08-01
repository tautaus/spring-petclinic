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
                    def dockerImage = docker.build("petclinic:latest", "--no-cache .")
                    echo "Docker Image built: ${dockerImage.id}"
                    env.DOCKER_IMAGE_ID = dockerImage.id
                }
            }
        }

        stage('Add SSH Key and Deploy Container to VM') {
            steps {
                script {
                    // Add the SSH key to known hosts and set permissions
                    sh '''
                        mkdir -p ~/.ssh
                        chmod 700 ~/.ssh
                        ssh-keyscan -H ec2-3-149-247-7.us-east-2.compute.amazonaws.com >> ~/.ssh/known_hosts
                        chmod 600 ~/.ssh/known_hosts
                        chmod 600 /var/jenkins_home/ansible.pem
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


//         stage('SonarQube analysis') {
//             steps {
//
//                 script {
//                     scannerHome = tool 'sonar-scanner'// must match the name of an actual scanner installation directory on your Jenkins build agent
//                 }
//                 withSonarQubeEnv('test_sonarqube') {
//                         sh """
//                             ${scannerHome}/bin/sonar-scanner \
//                             -Dproject.settings=sonar-project.properties
//                         """
//                 }
//
//             }
//         }

        stage('Run ZAP Scan') {
            steps {
                script {
                    // Pull the ZAP image
                    def zapImage = docker.image("ghcr.io/zaproxy/zaproxy:stable")

                    // Run the ZAP scan
                    zapImage.inside("-v ${WORKSPACE}/zap-report:/zap/wrk:rw --name zap-scan --rm") {
                        sh "zap-baseline.py -t http://jenkins:8080 -g gen.conf -r zap-report.html"
                    }
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
