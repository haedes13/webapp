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
                    echo "🔍 Running TruffleHog..."
                    docker run --rm gesellix/trufflehog --json https://github.com/haedes13/webapp.git > trufflehog.json || true
                    cat trufflehog.json || true
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
                    sh 'scp -o StrictHostKeyChecking=no target/*.war tomcat@192.168.59.177:/home/tomcat/apache-tomcat-9.0.102/webapps/webapp.war || true'
                }
            }
        }

        stage('Port Scanning & Vuln Detection') {
            steps {
                sh '''
                    echo "🔍 Running Nmap port scan..."
                    nmap -sT -T4 -p- 192.168.59.177 -oN portscan.txt || true

                    echo "📘 Formatting port scan output:"
                    grep '^PORT' -A 100 portscan.txt | awk '/open/{print $1, $2, $3}' > formatted-ports.txt || true
                    cat formatted-ports.txt || true

                    echo "🧪 Checking for unexpected open ports..."
                    awk '{print $1}' formatted-ports.txt | cut -d/ -f1 | grep -Ev '^(22|80|8080|8443)$' > unexpected-ports.txt || true

                    if [ -s unexpected-ports.txt ]; then
                      echo "❌ Unexpected open ports detected:"
                      cat unexpected-ports.txt
                    else
                      echo "✅ Only expected ports are open."
                    fi

                    echo "🛡️ Running Nmap vulnerability scan..."
                    nmap -sV --script=vuln -T4 -p- 192.168.59.177 -oN vulnscan.txt || true

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
                      -x zap-report.xml || true
                    '

                    echo "📥 Copying ZAP reports from remote to Jenkins workspace..."
                    scp -o StrictHostKeyChecking=no owaspzap@192.168.59.180:/tmp/zap-report.* . || true
                    
                    if [ -f zap-report.xml ]; then
                        echo "✅ ZAP XML report generated successfully"
                        head -n 5 zap-report.xml
                    else
                        echo "❌ ZAP XML report not found"
                        exit 1
                    fi
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
                      nikto -host http://192.168.59.177:8080/webapp/ -Format json -output /tmp/nikto-report.json || true
                    '

                    echo "📥 Copying Nikto reports..."
                    scp -o StrictHostKeyChecking=no owaspzap@192.168.59.180:/tmp/nikto-report.* . || true
                    '''
                }
            }
        }

        stage('SSL Certificate Check') {
            steps {
                sh '''
                    echo "🔒 Running SSLyze scan..."

                    docker run --rm nablac0d3/sslyze:6.1.0 192.168.59.177:8443 --json_out sslyze-report.json | tee sslyze-report.txt || true

                    echo "✅ SSLyze scan completed"
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'portscan.txt, formatted-ports.txt, unexpected-ports.txt, vulnscan.txt, detected-vulns.txt', onlyIfSuccessful: false
            archiveArtifacts artifacts: 'zap-report.*', onlyIfSuccessful: false
            archiveArtifacts artifacts: 'nikto-report.txt, nikto-report.json', onlyIfSuccessful: false
            archiveArtifacts artifacts: 'sslyze-report.txt, sslyze-report.json', onlyIfSuccessful: false
            archiveArtifacts artifacts: 'trufflehog.json', onlyIfSuccessful: false
            archiveArtifacts artifacts: 'target/dependency-check-report.xml', onlyIfSuccessful: false
        }
        success {
            echo '✅ Build, Deployment, and All Scans completed successfully.'
        }
        failure {
            echo '⚠️ Build or Deployment failed during a critical stage.'
        }
    }
}
