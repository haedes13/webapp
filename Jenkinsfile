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
                sh '''
                echo "🔐 Checking for secrets with TruffleHog..."
                rm -f trufflehog || true
                docker run --rm gesellix/trufflehog --json https://github.com/haedes13/webapp.git > trufflehog || true
                cat trufflehog || true
                '''
            }
        }

        stage('Source Composition Analysis') {
            steps {
                sh '''
                echo "📦 Running OWASP Dependency Check..."
                rm owasp* || true
                wget "https://raw.githubusercontent.com/haedes13/webapp/refs/heads/master/owasp-dependency-check.sh" || true
                chmod +x owasp-dependency-check.sh || true
                bash owasp-dependency-check.sh || true
                cat /var/lib/jenkins/OWASP-Dependency-Check/reports/dependency-check-report.xml || true
                '''
            }
        }

        stage('SAST') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                    echo "🔍 Running SonarQube scan..."
                    mvn sonar:sonar || true
                    cat target/sonar/report-task.txt || true
                    '''
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
                echo "🔍 Running Nmap port scan and vulnerability detection..."

                nmap -sT -T4 -p- 192.168.59.177 -oN portscan.txt || true

                echo "📘 Formatting port scan output:"
                grep '^PORT' -A 100 portscan.txt | awk '/open/{print $1, $2, $3}' > formatted-ports.txt || true
                cat formatted-ports.txt || true

                echo "🧪 Checking for unexpected open ports..."
                UNEXPECTED=$(awk '{print $1}' formatted-ports.txt | cut -d/ -f1 | grep -Ev '^(22|80|8080|8443)$' || true)

                if [ ! -z "$UNEXPECTED" ]; then
                  echo "⚠️ Unexpected open ports detected:"
                  echo "$UNEXPECTED"
                else
                  echo "✅ Only expected ports are open."
                fi

                echo "🛡️ Running Nmap vulnerability scan (non-root)..."
                nmap -sV --script=vuln -T4 -p- 192.168.59.177 -oN vulnscan.txt || true

                echo "📖 Extracting vulnerability summary..."
                grep -i "VULNERABLE" vulnscan.txt > detected-vulns.txt || true

                if [ -s detected-vulns.txt ]; then
                  echo "⚠️ Vulnerabilities found:"
                  cat detected-vulns.txt
                else
                  echo "✅ No known vulnerabilities found in scan."
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

                    echo "📥 Copying ZAP reports from remote..."
                    scp -o StrictHostKeyChecking=no owaspzap@192.168.59.180:/tmp/zap-report.* . || true
                    '''
                }
            }
        }

        stage('Nikto Scan') {
            steps {
                sshagent(['zap']) {
                    sh '''
                    echo "🔍 Running Nikto Scan..."

                    ssh -o StrictHostKeyChecking=no owaspzap@192.168.59.180 '
                      nikto -host http://192.168.59.177:8080/webapp/ -output /tmp/nikto-report.txt || true
                    '

                    echo "📥 Copying Nikto report..."
                    scp -o StrictHostKeyChecking=no owaspzap@192.168.59.180:/tmp/nikto-report.txt . || true
                    '''
                }
            }
        }

        stage('SSL Checks') {
            steps {
                sshagent(['zap']) {
                    sh '''
                    echo "🔐 Running SSL scan on port 8443..."

                    ssh -o StrictHostKeyChecking=no owaspzap@192.168.59.180 '
                      sslscan 192.168.59.177:8443 > /tmp/sslscan-report.txt || true
                    '

                    echo "📥 Copying SSL scan report..."
                    scp -o StrictHostKeyChecking=no owaspzap@192.168.59.180:/tmp/sslscan-report.txt . || true
                    '''
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'portscan.txt, formatted-ports.txt, vulnscan.txt, detected-vulns.txt', onlyIfSuccessful: false
            archiveArtifacts artifacts: 'zap-report.*', onlyIfSuccessful: false
            archiveArtifacts artifacts: 'nikto-report.txt', onlyIfSuccessful: false
            archiveArtifacts artifacts: 'sslscan-report.txt', onlyIfSuccessful: false
        }
        success {
            echo '✅ Build and Deployment completed successfully!'
        }
        failure {
            echo '❌ Build or Deployment failed!'
        }
    }
}
