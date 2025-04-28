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
                sh 'docker run --rm gesellix/trufflehog --json https://github.com/haedes13/webapp.git > trufflehog || true'
                sh 'cat trufflehog || true'
            }
        }

        stage('Source Composition Analysis') {
            steps {
                sh 'rm owasp* || true'
                sh 'wget "https://raw.githubusercontent.com/haedes13/webapp/refs/heads/master/owasp-dependency-check.sh" || true'
                sh 'chmod +x owasp-dependency-check.sh || true'
                sh 'bash owasp-dependency-check.sh || true'
                sh 'cat /var/lib/jenkins/OWASP-Dependency-Check/reports/dependency-check-report.xml || true'
            }
        }

        stage('SAST') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh 'mvn sonar:sonar || true'
                    sh 'cat target/sonar/report-task.txt || true'
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
                    echo "🔍 Running Nmap port scan and vulnerability detection on Tomcat server..."

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
                    '''
                }
            }
        }

        stage('Nikto Scan') {
            steps {
                sshagent(['zap']) {
                    sh '''
                    echo "🔍 Running Nikto Scan on Tomcat web application..."

                    ssh -o StrictHostKeyChecking=no owaspzap@192.16859.180 '
                      nikto -host http://192.168.59.177:8080/webapp/ -output /tmp/nikto-report.txt || true
                    '

                    echo "📥 Copying Nikto report from remote to Jenkins workspace..."
                    scp -o StrictHostKeyChecking=no owaspzap@192.168.59.180:/tmp/nikto-report.txt . || true
                    '''
                }
            }
        }

        stage('SSL Certificate Check') {
            steps {
                sh '''
                    echo "🔒 Running SSLyze scan on Tomcat server..."

                    docker run --rm nablac0d3/sslyze:6.1.0 192.168.59.177:8443 | tee sslyze-report.txt || true

                    echo "📄 SSLyze scan output saved to sslyze-report.txt"
                '''
            }
        }

        // New stage for uploading reports to Defect Dojo
        stage('Upload Reports to DefectDojo') {
            steps {
                sh '''
                    source /var/lib/jenkins/dojoenv/bin/activate
                    python3 upload_to_defectdojo.py zap-report.json ZAP Scan
                    python3 upload_to_defectdojo.py dependency-check-report.xml Dependency-Check Scan
                    python3 upload_to_defectdojo.py nikto-report.txt Nikto Scan
                    python3 upload_to_defectdojo.py sslyze-report.txt SSLyze Scan
                    deactivate
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'portscan.txt, formatted-ports.txt, unexpected-ports.txt, vulnscan.txt, detected-vulns.txt', onlyIfSuccessful: false
            archiveArtifacts artifacts: 'zap-report.*', onlyIfSuccessful: false
            archiveArtifacts artifacts: 'nikto-report.txt', onlyIfSuccessful: false
            archiveArtifacts artifacts: 'sslyze-report.txt', onlyIfSuccessful: false
        }
        success {
            echo '✅ Build, Deployment, and Security Scans completed successfully (with reports logged).'
        }
        failure {
            echo '⚠️ Build or Deployment failed during a critical non-security stage.'
        }
    }
}
