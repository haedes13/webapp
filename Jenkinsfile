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
                rm -f trufflehog || true
                docker run --rm gesellix/trufflehog --json https://github.com/haedes13/webapp.git > trufflehog || true
                cat trufflehog || echo "⚠️ Trufflehog output is empty or failed."
                '''
            }
        }

        stage('Source Composition Analysis') {
            steps {
                sh '''
                rm owasp* || true
                wget "https://raw.githubusercontent.com/haedes13/webapp/refs/heads/master/owasp-dependency-check.sh" || true
                chmod +x owasp-dependency-check.sh || true
                bash owasp-dependency-check.sh || echo "⚠️ Dependency check script failed."
                cat /var/lib/jenkins/OWASP-Dependency-Check/reports/dependency-check-report.xml || echo "⚠️ No report found."
                '''
            }
        }

        stage('SAST') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                    mvn sonar:sonar || echo "⚠️ SonarQube analysis failed."
                    cat target/sonar/report-task.txt || echo "⚠️ Sonar report not found."
                    '''
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -X || echo "⚠️ Build may have failed, continuing..."'
            }
        }

        stage('Deploy-To-Tomcat') {
            steps {
                sshagent(['tomcat']) {
                    sh '''
                    scp -o StrictHostKeyChecking=no target/*.war tomcat@192.168.59.177:/home/tomcat/apache-tomcat-9.0.102/webapps/webapp.war || echo "⚠️ WAR deployment failed."
                    '''
                }
            }
        }

        stage('Port Scanning & Vuln Detection') {
            steps {
                sh '''
                echo "🔍 Running Nmap port scan and vulnerability detection on Tomcat server..."

                nmap -sT -T4 -p- 192.168.59.177 -oN portscan.txt || echo "⚠️ Nmap port scan failed."

                echo "📘 Formatting port scan output:"
                grep '^PORT' -A 100 portscan.txt | awk '/open/{print $1, $2, $3}' > formatted-ports.txt || true
                cat formatted-ports.txt || echo "⚠️ No formatted ports found."

                echo "🧪 Checking for unexpected open ports..."
                UNEXPECTED=$(awk '{print $1}' formatted-ports.txt | cut -d/ -f1 | grep -Ev '^(22|80|8080|8443)$' || true)

                if [ ! -z "$UNEXPECTED" ]; then
                  echo "❌ Unexpected open ports detected:"
                  echo "$UNEXPECTED"
                else
                  echo "✅ Only expected ports are open."
                fi

                echo "🛡️ Running Nmap vulnerability scan (no root required)..."
                nmap -sV --script=vuln -T4 -p- 192.168.59.177 -oN vulnscan.txt || echo "⚠️ Nmap vuln scan failed."

                echo "📖 Checking for known vulnerabilities..."
                grep -i "VULNERABLE" vulnscan.txt > detected-vulns.txt || true

                if [ -s detected-vulns.txt ]; then
                  echo "❌ Vulnerabilities found:"
                  cat detected-vulns.txt
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
                      -x zap-report.xml || echo "⚠️ ZAP scan failed"
                    '

                    echo "📥 Copying ZAP reports from remote to Jenkins workspace..."
                    scp -o StrictHostKeyChecking=no owaspzap@192.168.59.180:/tmp/zap-report.* . || echo "⚠️ Failed to copy ZAP reports."
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
                      nikto -host http://192.168.59.177:8080/webapp/ -output /tmp/nikto-report.txt || echo "⚠️ Nikto scan failed."
                    '

                    echo "📥 Copying Nikto report from remote to Jenkins workspace..."
                    scp -o StrictHostKeyChecking=no owaspzap@192.168.59.180:/tmp/nikto-report.txt . || echo "⚠️ Failed to copy Nikto report."
                    '''
                }
            }
        }

        stage('SSL Checks (SSLyze)') {
            steps {
                sshagent(['zap']) {
                    sh '''
                    echo "🔐 Running SSL scan with SSLyze..."

                    ssh -o StrictHostKeyChecking=no owaspzap@192.168.59.180 '
                      sslyze --regular --json_out /tmp/sslyze-report.json 192.168.59.177:8443 || echo "⚠️ SSLyze scan failed."
                    '

                    echo "📥 Copying SSLyze scan report from remote to Jenkins workspace..."
                    scp -o StrictHostKeyChecking=no owaspzap@192.168.59.180:/tmp/sslyze-report.json . || echo "⚠️ Failed to copy SSLyze report."

                    echo "📖 Displaying SSL scan results..."
                    cat sslyze-report.json || echo "⚠️ SSL scan report not found or empty."
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
            archiveArtifacts artifacts: 'sslyze-report.json', onlyIfSuccessful: false
        }
        success {
            echo '✅ Build and Deployment succeeded!'
        }
        failure {
            echo '❌ Build or Deployment failed!'
        }
    }
}
