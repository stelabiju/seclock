pipeline{
    agent any

    environment {
        image_name = "seclock"
    }

    stages {
        stage ('Checkout') {
            steps {
                checkout scm
            }
        }

        stage ('Semgrep') {
            steps {
                sh '''
                docker run --rm -v "$WORKSPACE:/src" semgrep/semgrep semgrep scan --config auto /src
                '''
            }
        }

        stage ('SCA') {
            steps {
                sh '''
                docker run --rm -v "$WORKSPACE:/src" aquasec/trivy:latest fs --scanners vuln /src
                '''
            }
        }

        stage ('Docker Build') {
            steps {
                sh '''
                docker build -t $IMAGE_NAME:V1 .
                '''
            }
        }
        stage ('Trivy Image Scan') {
            steps {
                sh '''
                docker run --rm aquasec/trivy:latest $IMAGE_NAME:V1 
                ''' 
            }
        }

    }
}