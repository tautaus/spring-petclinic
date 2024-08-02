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

        stage('Push Docker Image to DockerHub') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-credentials') {
                        def dockerImage = docker.image("chariot97/petclinic:latest")
                        dockerImage.push('latest')
                    }
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
                    // Create the zap-report directory
                    sh "mkdir -p ${WORKSPACE}/zap-report"

                    // Pull the ZAP image
                    def zapImage = docker.image("ghcr.io/zaproxy/zaproxy:stable")

                    // Run the ZAP scan
                    try {
                    // -I to ignore errors
                    // -j to include inline css
                        zapImage.inside("-v ${WORKSPACE}/zap-report:/zap/wrk:rw --name zap-scan --rm") {
                            sh "zap-baseline.py -t http://3.149.247.7:8080 -g gen.conf -I -j -r zap-report.html"
                            // Ensure the report is copied to the mounted volume
                            sh "cp /zap/wrk/zap-report.html ${WORKSPACE}/zap-report/"
                        }

                        // Add a small delay before container cleanup
                        sleep 5

                        // Debug steps
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
            publishHTML(target: [
                allowMissing: false,
                alwaysLinkToLastBuild: false,
                keepAll: true,
                reportDir: "${WORKSPACE}/zap-report",
                reportFiles: 'zap-report.html',
                reportName: 'OWASP_ZAP_Report'
            ])

             // Archive the ZAP report as a downloadable artifact
            archiveArtifacts artifacts: 'zap-report/*.html', fingerprint: true
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}

