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
                    git url: 'https://github.com/tautaus/spring-petclinic.git', branch: 'FinalProject_main', credentialsId: 'github-token'
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


        stage('SonarQube analysis') {
            steps {

                script {
                    scannerHome = tool 'sonar-scanner'// must match the name of an actual scanner installation directory on your Jenkins build agent
                }
                withSonarQubeEnv('test_sonarqube') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dproject.settings=sonar-project.properties
                        """
                }

            }
        }

        stage('Run ZAP Scan') {
            steps {
                script {
                    // Create the report directory
                    sh "mkdir -p ${WORKSPACE}/zap-report"

                    // Pull the ZAP image
                    def zapImage = docker.image("ghcr.io/zaproxy/zaproxy:stable")

                    // Run the ZAP scan
                    try {
                        zapImage.inside("-v ${WORKSPACE}/zap-report:/zap/wrk:rw --name zap-scan --rm") {
                            sh "zap-baseline.py -t http://3.149.247.7:8080 -g gen.conf -I -r zap-report.html"
                            // List files in the container to check if the report was generated
                            sh "ls -la /zap/wrk"
                        }

                        // Debug steps to verify the report in the workspace
                        sh "ls -la ${WORKSPACE}/zap-report"
                        sh "cat ${WORKSPACE}/zap-report/zap-report.html || echo 'ZAP report not found'"

                    } catch (Exception e) {
                        echo "Warnings found during ZAP scan: ${e.message}"
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                // Stopping and removing the Docker image if exists
                if (env.DOCKER_IMAGE_ID) {
                    echo "Stopping and removing Docker Image with ID: ${env.DOCKER_IMAGE_ID}"
                    sh "docker rmi -f ${env.DOCKER_IMAGE_ID}"
                }
            }
        }
        success {
            echo 'Pipeline completed successfully!'
            // Verify the report exists before publishing
            script {
                if (fileExists("${WORKSPACE}/zap-report/zap-report.html")) {
                    publishHTML([
                        reportName: 'ZAP_HTML_Report',
                        reportDir: 'zap-report', // Directory where the report is generated
                        reportFiles: 'zap-report.html', // Report file name(s)
                        keepAll: true, // Keep all reports (useful for historical comparisons)
                        alwaysLinkToLastBuild: true, // Always link to the last build's report
                        allowMissing: false // Fail the build if the report is missing
                    ])
                } else {
                    echo 'ZAP report not found, skipping HTML report publishing.'
                    currentBuild.result = 'UNSTABLE' // Mark the build as unstable if the report is missing
                }
            }
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}
