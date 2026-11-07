pipeline {
    agent any

    environment {
        DOCKER_USER = "rajashreem14"
        IMAGE_NAME  = "${DOCKER_USER}/react-cicd-app"
        IMAGE_TAG   = "${env.BUILD_NUMBER}"
        AWS_REGION  = "ap-south-1"
        EKS_CLUSTER = "react-eks-cluster"
    }

    stages {
        stage('1. Checkout & Clean') {
            steps {
                cleanWs()
                checkout scm
            }
        }

        stage('2. Install Dependencies') {
            steps {
                bat 'npm ci'
            }
        }

        stage('3. Unit Testing') {
            steps {
                bat 'npm test -- --run'
            }
        }

        stage('4. Lint') {
            steps {
                bat 'npm run lint'
            }
        }

        stage('5. Build React App') {
            steps {
                bat 'npm run build'
            }
        }

        stage('6. SonarQube Code Scan') {
            steps {
                withSonarQubeEnv('SonarQubeServer') {
                    bat 'npx sonarqube-scanner'
                }
            }
        }

        stage('7. Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('8. Docker Build') {
            steps {
                bat "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
                bat "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest"
            }
        }

        stage('9. Trivy Image Scan') {
            steps {
                bat "trivy image --exit-code 1 --severity CRITICAL --no-progress ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('10. Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    bat 'docker login -u %USER% -p %PASS%'
                    bat "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                    bat "docker push ${IMAGE_NAME}:latest"
                }
            }
        }

        stage('11. Connect to EKS') {
            steps {
                withCredentials([string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'), string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')]) {
                    bat "aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER}"
                }
            }
        }

        stage('12. Deploy to Kubernetes') {
            steps {
                withCredentials([string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'), string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')]) {
                    bat "powershell -Command \"(Get-Content k8s\\deployment.yaml) -replace 'rajashreem14/react-cicd-app:IMAGE_TAG', '${IMAGE_NAME}:${IMAGE_TAG}' | Set-Content k8s\\deployment.yaml\""
                    bat 'kubectl apply -f k8s/'
                }
            }
        }

        stage('13. Verify Rollout & Health Check') {
            steps {
                withCredentials([string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'), string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')]) {
                    bat 'kubectl rollout status deployment/react-cicd-app --timeout=180s'
                    bat 'kubectl get pods -l app=react-cicd-app'
                    bat 'kubectl get svc react-cicd-service'
                }
            }
        }
    }
}

