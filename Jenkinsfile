pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/haedes13/webapp.git'
            }
        }

        stage('Initialize') {
            steps {
                echo "PATH = ${env.PATH}"
                echo "M2_HOME = ${env.M2_HOME}"
            }
        }

        stage('Check-Git-Secrets') { 
            steps { 
                sh 'rm -f trufflehog || true'
                sh 'docker run --rm gesellix/trufflehog --json https://github.com/haedes13/webapp.git > trufflehog'
                sh 'cat trufflehog'
            }
        }

        stage('Source Composition Analysis') {
            steps {
                sh 'rm owasp* || true'
                sh 'wget "https://raw.githubusercontent.com/haedes13/webapp/refs/heads/master/owasp-dependency-check.sh"'
                sh 'chmod +x owasp-dependency-check.sh'
                sh 'bash owasp-dependency-check.sh'
                sh 'cat /var/lib/jenkins/OWASP-Dependency-Check/reports/dependency-check-report.xml'
            }
        }

        stage('SAST') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh 'mvn sonar:sonar'
                    sh 'cat target/sonar/report-task.txt'
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -X'
            }
        }

        stage('Deploy-To-Tomcat') {
            steps {
                sshagent(['tomcat']) { 
                    sh 'scp -o StrictHostKeyChecking=no target/*.war tomcat@192.168.59.177:/home/tomcat/apache-tomcat-9.0.102/webapps/webapp.war'
                }
            }
        }

        stage('Port Scanning & Vuln Detection') {
            steps {
                sh '''
                    echo "🔍 Running Nmap port scan and vulnerability detection on Tomcat server..."

                    nmap -sT -T4 -p- 192.168.59.177 -oN portscan.txt

                    echo "📘 Formatting port scan output:"
                    grep '^PORT' -A 100 portscan.txt | awk '/open/{print $1, $2, $3}' > formatted-ports.txt
                    cat formatted-ports.txt

                    echo "🧪 Checking for unexpected open ports..."
                    UNEXPECTED=$(awk '{print $1}' formatted-ports.txt | cut -d/ -f1 | grep -Ev '^(22|80|8080)$' || true)

                    if [ ! -z "$UNEXPECTED" ]; then
                      echo "❌ Unexpected open ports detected:"
                      echo "$UNEXPECTED"
                      exit 1
                    else
                      echo "✅ Only expected ports are open."
                    fi

                    echo "🛡️ Running Nmap vulnerability scan (no root required)..."
                    nmap -sV --script=vuln -T4 -p- 192.168.59.177 -oN vulnscan.txt

                    echo "📖 Checking for known vulnerabilities..."
                    grep -i "VULNERABLE" vulnscan.txt > detected-vulns.txt || true

                    if [ -s detected-vulns.txt ]; then
                      echo "❌ Vulnerabilities found:"
                      cat detected-vulns.txt
                      exit 1
                    else
                      echo "✅ No known vulnerabilities found."
                    fi
                '''
            }
        }

        stage('DAST') {
            steps {
                sshagent(['zap']) {
                    sh '''
                    echo "⚡ Running OWASP ZAP Baseline Scan..."

                    ssh -o StrictHostKeyChecking=no owaspzap@192.168.59.180 '
                      docker run -v /tmp:/zap/wrk/:rw -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
                      -t http://192.168.59.177:8080/webapp/ \
                      -r zap-report.html \
                      -J zap-report.json \
                      -x zap-report.xml || true
                    '

                    echo "📥 Copying ZAP reports from remote to Jenkins workspace..."
                    scp -o StrictHostKeyChecking=no owaspzap@192.168.59.180:/tmp/zap-report.* .
                    '''
                }
            }
        }

        stage('Nikto Scan') {
            steps {
                sshagent(['zap']) {
                    sh '''
                    echo "🔍 Running Nikto Scan on Tomcat web application..."

                    ssh -o StrictHostKeyChecking=no owaspzap@192.168.59.180 '
                      nikto -host http://192.168.59.177:8080/webapp/ -output /tmp/nikto-report.txt || true
                    '

                    echo "📥 Copying Nikto report from remote to Jenkins workspace..."
                    scp -o StrictHostKeyChecking=no owaspzap@192.168.59.180:/tmp/nikto-report.txt .
                    '''
                }
            }
        }

    post {
        always {
            archiveArtifacts artifacts: 'portscan.txt, formatted-ports.txt, vulnscan.txt, detected-vulns.txt', onlyIfSuccessful: false
            archiveArtifacts artifacts: 'zap-report.*', onlyIfSuccessful: false
            archiveArtifacts artifacts: 'nikto-report.txt', onlyIfSuccessful: false
            archiveArtifacts artifacts: 'fuzz-report.*', onlyIfSuccessful: false
        }
        success {
            echo '✅ Build and Deployment succeeded!'
        }
        failure {
            echo '❌ Build or Deployment failed!'
        }
    }
}
