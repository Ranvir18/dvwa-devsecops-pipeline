pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner'

                    withSonarQubeEnv('SonarQube') {
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
                }
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh '''
                    /usr/bin/trivy fs \
                    --severity HIGH,CRITICAL \
                    --format table \
                    -o trivy-report.txt \
                    .
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivy-report.txt', fingerprint: true
        }
    }
}
