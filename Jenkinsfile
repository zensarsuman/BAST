pipeline { 
agent any

environment {
    TRIVY_VERSION = "0.51.1"
    DOCKER_IMAGE = "bast-app"
    TARGET_URL = "http://your-dapp-url.com"
    CONTRACTS_DIR = "contracts/"
    SONAR_PROJECT_KEY = "bast-project"
    SONAR_HOST_URL = "http://your-sonarqube-server"
    SONAR_TOKEN = credentials('sonar-token')
}

stages {
    stage('Checkout Code') {
        steps {
            checkout scm
        }
    }

    stage('SonarQube Scan') {
        steps {
            withSonarQubeEnv('MySonarQube') {
                script {
                    def isNode = fileExists('package.json')
                    def isGo = fileExists('go.mod')

                    if (isNode || isGo) {
                        echo "Running SonarQube scanner for ${isNode ? 'Node.js' : 'Go'} project."
                        sh '''
                            sonar-scanner \
                              -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                              -Dsonar.sources=. \
                              -Dsonar.host.url=${SONAR_HOST_URL} \
                              -Dsonar.login=${SONAR_TOKEN}
                        '''
                    } else {
                        echo "SonarQube scan skipped: neither Node.js nor Go project detected."
                    }
                }
            }
        }
    }

    stage('Detect Contract Type and Run SAST') {
        steps {
            script {
                def solidityFiles = sh(
                    script: "find ${CONTRACTS_DIR} -name '*.sol' | wc -l",
                    returnStdout: true
                ).trim()

                if (solidityFiles.toInteger() > 0) {
                    echo "Detected Solidity smart contracts. Running BAST-Blockchain-Eth-Code-Scanner..."
                    sh '''
                        pip install slither-analyzer
                        slither ${CONTRACTS_DIR} --json slither-report.json || true
                    '''
                    archiveArtifacts artifacts: 'slither-report.json', fingerprint: true
                } else {
                    echo "No Solidity found. Running BAST-Blockchain-Chain-Code-Scanner..."
                    sh '''
                        go install github.com/hyperledger-labs/fabric-smart-contract-analyzer@latest
                        fabric-smart-contract-analyzer --path ${CONTRACTS_DIR} --output chaincode-report.json || true
                    '''
                    archiveArtifacts artifacts: 'chaincode-report.json', fingerprint: true
                }
            }
        }
    }

    stage('Dependency Scan - npm / pip / maven / docker / config files') {
        steps {
            sh '''
                # NPM scan
                if [ -f package.json ]; then
                  npm install --omit=dev || true
                  npm audit --json > npm-audit-report.json || true
                fi

                # Python scan
                if [ -f requirements.txt ]; then
                  pip install -r requirements.txt || true
                  pip-audit -r requirements.txt --format=json -o pip-audit-report.json || true
                fi

                # Maven scan
                if [ -f pom.xml ]; then
                  mvn org.owasp:dependency-check-maven:check || true
                  cp target/dependency-check-report.json maven-audit-report.json || true
                fi

                # Dockerfile scan
                if [ -f Dockerfile ]; then
                  trivy fs --format json --output dockerfile-audit-report.json . || true
                fi

                # YAML scan
                find . -type f -name '*.yaml' -o -name '*.yml' | while read yamlfile; do
                  trivy config --format json --output="${yamlfile}-config-audit.json" "$yamlfile" || true
                done
            '''
        }
        post {
            always {
                archiveArtifacts artifacts: '**/*audit-report.json,**/dependency-check-report.json,**/*-config-audit.json', fingerprint: true
            }
        }
    }

    stage('Dynamic Analysis - OWASP ZAP') {
        steps {
            sh '''
                docker run -t owasp/zap2docker-stable zap-baseline.py \
                  -t ${TARGET_URL} -g gen.conf -r zap-report.html || true
            '''
        }
        post {
            always {
                archiveArtifacts artifacts: 'zap-report.html', fingerprint: true
            }
        }
    }

    stage('Build Docker Image') {
        steps {
            sh 'docker build -t ${DOCKER_IMAGE} .'
        }
    }

    stage('Infrastructure Scan - Trivy') {
        steps {
            sh '''
                wget https://github.com/aquasecurity/trivy/releases/download/v${TRIVY_VERSION}/trivy_${TRIVY_VERSION}_Linux-64bit.deb
                sudo dpkg -i trivy_${TRIVY_VERSION}_Linux-64bit.deb
                trivy image ${DOCKER_IMAGE} --format json --output trivy-report.json || true
            '''
        }
        post {
            always {
                archiveArtifacts artifacts: 'trivy-report.json', fingerprint: true
            }
        }
    }

    stage('Summary') {
        steps {
            echo "BAST pipeline complete. All reports archived for review."
        }
    }
}

post {
    always {
        echo "Pipeline completed. Review static, dynamic, infra, dependency, and config scan outputs."
    }
}

}
