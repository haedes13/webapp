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
                sh 'docker run --rm gesellix/trufflehog --json https://github.com/haedes13/webapp.git > trufflehog.json || true'
                sh 'cat trufflehog.json || true'
            }
        }

        stage('Source Composition Analysis') {
            steps {
                sh 'rm owasp* || true'
                sh 'wget "https://raw.githubusercontent.com/haedes13/webapp/refs/heads/master/owasp-dependency-check.sh" || true'
                sh 'chmod +x owasp-dependency-check.sh || true'
                sh 'bash owasp-dependency-check.sh || true'
                sh 'cat /var/lib/jenkins/OWASP-Dependency-Check/reports/dependency-check-report.json || true'
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

                    nmap -sT -T4 -p- 192.168.59.177 -oX portscan.xml || true
                    echo "📘 Formatting port scan output to JSON..."
                    python3 -c 'import xml.etree.ElementTree as ET, json; tree = ET.parse("portscan.xml"); root = tree.getroot(); nmap_json = {"scan": {} }; nmap_json["scan"]["192.168.59.177"] = {"host": {"address": "192.168.59.177", "ports": []}}; for port in root.findall(".//port"): nmap_json["scan"]["192.168.59.177"]["host"]["ports"].append({"port": port.attrib["portid"], "state": port.find("state").attrib["state"]}); print(json.dumps(nmap_json))' > portscan.json || true
                    cat portscan.json || true

                    echo "🧪 Checking for unexpected open ports..."
                    awk '{print $1}' formatted-ports.txt | cut -d/ -f1 | grep -Ev '^(22|80|8080|8443)$' > unexpected-ports.txt || true

                    if [ -s unexpected-ports.txt ]; then
                      echo "❌ Unexpected open ports detected:"
                      cat unexpected-ports.txt
                    else
                      echo "✅ Only expected ports are open."
                    fi

                    echo "🛡️ Running Nmap vulnerability scan..."
                    nmap -sV --script=vuln -T4 -p- 192.168.59.177 -oX vulnscan.xml || true
                    python3 -c 'import xml.etree.ElementTree as ET, json; tree = ET.parse("vulnscan.xml"); root = tree.getroot(); vuln_json = {"scan": {} }; vuln_json["scan"]["192.168.59.177"] = {"host": {"address": "192.168.59.177", "vulnerabilities": []}}; for vuln in root.findall(".//script[@id='vuln']"): vuln_json["scan"]["192.168.59.177"]["host"]["vulnerabilities"].append({"vuln_id": vuln.attrib["id"], "output": vuln.find("output").text}); print(json.dumps(vuln_json))' > vulnscan.json || true
                    cat vulnscan.json || true
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
                    scp -o StrictHostKeyChecking=no owaspzap@192.168.59.180:/tmp/zap-report.json . || true
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
                      nikto -host http://192.168.59.177:8080/webapp/ -output /tmp/nikto-report.json -Format json || true
                    '

                    echo "📥 Copying Nikto report from remote to Jenkins workspace..."
                    scp -o StrictHostKeyChecking=no owaspzap@192.168.59.180:/tmp/nikto-report.json . || true
                    '''
                }
            }
        }

        stage('SSL Certificate Check') {
            steps {
                sh '''
                    echo "🔒 Running SSLyze scan on Tomcat server..."

                    docker run --rm nablac0d3/sslyze:6.1.0 192.168.59.177:8443 | tee sslyze-report.json || true

                    echo "📄 SSLyze scan output saved to sslyze-report.json"
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'portscan.json, vulnscan.json, zap-report.json, nikto-report.json, sslyze-report.json', onlyIfSuccessful: false
        }
        success {
            echo '✅ Build, Deployment, and Security Scans completed successfully (with reports logged).'
        }
        failure {
            echo '⚠️ Build or Deployment failed during a critical non-security stage.'
        }
    }
}
