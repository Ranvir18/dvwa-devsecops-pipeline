pipeline {
    agent any

    environment {
        SONAR_URL = "http://localhost:9000"
        PROJECT_KEY = "DVWA"
    }

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

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh '''
                export LANG=en_US.UTF-8
                export LC_ALL=en_US.UTF-8

                /usr/bin/trivy fs \
                --severity HIGH,CRITICAL \
                --format table \
                -o trivy-report.txt \
                .
                '''
            }
        }

        stage('Fetch SonarQube Report') {
            steps {
                withCredentials([string(credentialsId: 'sonarqube-token', variable: 'TOKEN')]) {

                    script {

                        def response = sh(
                            script: """
                            curl -s -u \$TOKEN: \
                            "${SONAR_URL}/api/measures/component?component=${PROJECT_KEY}&metricKeys=bugs,vulnerabilities,code_smells,security_hotspots,coverage,duplicated_lines_density,reliability_rating,security_rating,sqale_rating"
                            """,
                            returnStdout: true
                        ).trim()

                        writeFile file: 'sonarqube-report.json', text: response
                    }
                }
            }
        }

    }

    post {

        always {

            archiveArtifacts artifacts: 'trivy-report.txt', fingerprint: true
            archiveArtifacts artifacts: 'sonarqube-report.json', fingerprint: true

            script {

                def sonar = readJSON file: 'sonarqube-report.json'

                def metrics = [:]

                sonar.component.measures.each {
                    metrics[it.metric] = it.value
                }

                emailext(

                    to: "ranvirsh05@gmail.com",

                    subject: "[DevSecOps] Security Scan Report | ${env.JOB_NAME} #${env.BUILD_NUMBER}",

                    mimeType: 'text/html',

                    attachmentsPattern: 'trivy-report.txt',

                    body: """
<html>

<body style="font-family:Arial">

<h2>DevSecOps Security Scan Report</h2>

<p>Hello Team,</p>

<p>
The automated security assessment has completed successfully.
Below is the SonarQube analysis summary.
The detailed Trivy vulnerability report is attached.
</p>

<hr>

<h3>Build Details</h3>

<ul>
<li><b>Project:</b> ${env.JOB_NAME}</li>
<li><b>Build Number:</b> ${env.BUILD_NUMBER}</li>
<li><b>Status:</b> ${currentBuild.currentResult}</li>
</ul>

<hr>

<h3>SonarQube Summary</h3>

<table border="1" cellpadding="6">

<tr><td><b>Bugs</b></td><td>${metrics['bugs']}</td></tr>

<tr><td><b>Vulnerabilities</b></td><td>${metrics['vulnerabilities']}</td></tr>

<tr><td><b>Code Smells</b></td><td>${metrics['code_smells']}</td></tr>

<tr><td><b>Security Hotspots</b></td><td>${metrics['security_hotspots']}</td></tr>

<tr><td><b>Coverage</b></td><td>${metrics['coverage']}%</td></tr>

<tr><td><b>Duplicated Lines</b></td><td>${metrics['duplicated_lines_density']}%</td></tr>

<tr><td><b>Reliability Rating</b></td><td>${metrics['reliability_rating']}</td></tr>

<tr><td><b>Security Rating</b></td><td>${metrics['security_rating']}</td></tr>

<tr><td><b>Maintainability Rating</b></td><td>${metrics['sqale_rating']}</td></tr>

</table>

<br>

<p>
Please review the attached Trivy report and address any findings before deployment.
</p>

<br>

Regards,<br>

<b>DevSecOps Team</b>

</body>

</html>
"""
                )
            }
        }
    }
}
