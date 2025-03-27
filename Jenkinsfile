pipeline {
    agent any

    tools {
        maven 'maven'
    }

    environment {
        NEXUS_URL = 'http://172.212.80.198:8081/repository/maven-releases/'
        NEXUS_CREDENTIALS_ID = 'nexus-cred'
        SONARQUBE_URL = 'http://20.42.94.196:9000'
        APP_NAME = "database_service_project"
        VERSION = "0.0.${BUILD_NUMBER}"  
        REPO_URL = "http://172.212.80.198:8081/repository/maven-releases/"
        IMAGE_TAG = "${BUILD_NUMBER}" // Fixed Docker tag format
        AWS_REGION = "us-east-2"
        ECR_REPO = 'sai/jp-morgan'
        AWS_ACCOUNT_ID = '590183994092'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/iamnothum/Boardgame.git'
            }
        }

        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }

        stage('Test') {
            steps {
                sh "mvn test"
            }
        }

        stage('Install Trivy') {
            steps {
                script {
                    sh """
                        if ! command -v trivy &> /dev/null; then
                            echo "Installing Trivy..."
                            curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh
                            mv ./bin/trivy /usr/local/bin/trivy
                            chmod +x /usr/local/bin/trivy
                        fi
                    """
                    sh "trivy --version"
                }
            }
        }

        stage('Trivy SBOM & SCA Scan') {
            steps {
                script {
                    sh """
                        trivy fs --format cyclonedx -o sbom.json .
                    """
                    sh """
                        curl -u admin:password --upload-file sbom.json ${NEXUS_URL}/sbom/sbom-report.json
                    """
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                        sh """
                            mvn sonar:sonar \
                                -Dsonar.projectKey=Boardgame \
                                -Dsonar.host.url=${SONARQUBE_URL} \
                                -Dsonar.login=${SONAR_TOKEN}
                        """
                    }
                }
            }
        }

        // stage('SonarQube Quality Gate') {
        //     steps {
        //         script {
        //             timeout(time: 5, unit: 'MINUTES') { // Wait up to 5 minutes for results
        //                 def qualityGate = waitForQualityGate()
        //                 if (qualityGate.status != 'OK') {
        //                     error "SonarQube Quality Gate failed: ${qualityGate.status}"
        //                 }
        //             }
        //         }
        //     }
        // }

        stage('Build') {
            steps {
                sh "mvn clean package"
            }
        }

        stage('Build & Push JAR to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'nexus-id', maven: 'maven') {
                    sh """
                        mvn deploy \
                        -DaltDeploymentRepository="maven-releases::default::${NEXUS_URL}" \
                        -Drevision=0.0.${BUILD_NUMBER}
                    """
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build -t myrepo/database_service:${IMAGE_TAG} .
                """
            }
        }

        // stage('Trivy Scan for Docker Image') {
        //     steps {
        //         script {
        //             def scanResult = sh(script: """
        //                 trivy image --exit-code 1 --severity CRITICAL myrepo/database_service:${IMAGE_TAG} || echo "VULNERABILITIES_FOUND"
        //             """, returnStdout: true).trim()

        //             if (scanResult.contains("VULNERABILITIES_FOUND")) {
        //                 error "Critical vulnerabilities found in Docker image! Aborting build."
        //             }
        //         }
        //     }
        // }

        stage('Install AWS CLI') {
            steps {
                script {
                    sh """
                        if ! command -v aws &> /dev/null; then
                            echo "AWS CLI not found! Installing..."
                            apt update && apt install -y curl unzip awscli
                        fi
                        aws --version
                    """
                }
            }
        }
 
        stage('Login to AWS ECR & Push Image') {
            steps {
                withAWS(credentials: 'aws-cred', region: "${AWS_REGION}") {
                    script {
                        def ECR_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
                        def IMAGE_NAME = "database_service"
                        def IMAGE_TAG = "${BUILD_NUMBER}"

                        sh """
                            echo "Logging into AWS ECR..."
                            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_URI}

                            echo "Tagging Docker image..."
                            docker tag myrepo/${IMAGE_NAME}:${IMAGE_TAG} ${ECR_URI}:${IMAGE_TAG}

                            echo "Pushing Docker image..."
                            docker push ${ECR_URI}:${IMAGE_TAG}
                        """
                    }
                }
            }
        }
    }
}
