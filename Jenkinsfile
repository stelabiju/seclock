pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '457660516559'
        ECR_REPO_URI = '457660516559.dkr.ecr.us-east-1.amazonaws.com/seclock'
        EKS_CLUSTER_NAME = 'seclock-cluster'
        K8S_NAMESPACE = 'seclock-prod'
    }

    stages {

        stage('1. Checkout') {
            steps {
                checkout scm
            }
        }

        stage('2. Gitleaks') {
            steps {
                sh '''
                docker run --rm \
                  -v "$WORKSPACE:/path" \
                  zricethezav/gitleaks:latest \
                  detect --source /path --redact -v || true
                '''
            }
        }

        stage('3. Semgrep SAST') {
            steps {
                sh '''
                docker run --rm \
                  -v "$WORKSPACE:/src" \
                  semgrep/semgrep \
                  semgrep scan --config auto /src
                '''
            }
        }

        stage('4. Trivy SCA') {
            steps {
                sh '''
                docker run --rm \
                  -v "$WORKSPACE:/src" \
                  aquasec/trivy:latest \
                  fs --scanners vuln /src
                '''
            }
        }

        stage('5. Docker Build') {
            steps {
                sh '''
                docker build -t seclock:${BUILD_NUMBER} .
                '''
            }
        }

        stage('6. Trivy Image Scan') {
            steps {
                sh '''
                docker run --rm \
                  -v /var/run/docker.sock:/var/run/docker.sock \
                  aquasec/trivy:latest \
                  image seclock:${BUILD_NUMBER}
                '''
            }
        }

        stage('7. ECR Login & Push') {
            steps {
                sh '''
                aws ecr get-login-password --region ${AWS_REGION} | \
                docker login --username AWS --password-stdin \
                ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                docker tag seclock:${BUILD_NUMBER} \
                ${ECR_REPO_URI}:${BUILD_NUMBER}

                docker push ${ECR_REPO_URI}:${BUILD_NUMBER}
                '''
            }
        }

        stage('8. Update Kubernetes Manifest') {
            steps {
                sh '''
                sed -i "s|image:.*|image: ${ECR_REPO_URI}:${BUILD_NUMBER}|" \
                k8s/deployment.yaml
                '''
            }
        }

    }

    post {
        always {
            echo 'Cleaning up workspace...'
            deleteDir()
        }

        failure {
            echo 'Pipeline failed. Check the Console Output for the failed stage.'
        }
    }
}