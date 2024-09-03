pipeline {
    agent any
    
    environment {
        VERSION = "${env.BUILD_ID}"
        AWS_DEFAULT_REGION = "us-east-1"
        IMAGE_REPO_NAME = "image-repo"
        IMAGE_TAG = "${env.BUILD_ID}"
        REPOSITORY_URI = "471112823225.dkr.ecr.us-east-1.amazonaws.com/image-repo"
    }

    stages {
        stage('Build with Maven') {
            steps {
                sh 'cd SampleWebApp && mvn clean install'
            }
        }

        stage('Test') {
            steps {
                sh 'cd SampleWebApp && mvn test'
            }
        }

        stage('Code Quality Scan') {
            steps {
                withSonarQubeEnv('sonar-scanner') {
                    sh "mvn -f SampleWebApp/pom.xml sonar:sonar"
                }
            }
        }

        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: true
            }
        }

        stage('Login to AWS ECR') {
            steps {
                withCredentials([string(credentialsId: 'aws_access_key_id', variable: 'AWS_ACCESS_KEY_ID'),
                                 string(credentialsId: 'aws_secret_access_key', variable: 'AWS_SECRET_ACCESS_KEY')]) {
                    script {
                        sh 'aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID'
                        sh 'aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY'
                        sh 'aws configure set region ${AWS_DEFAULT_REGION}'
                        sh """aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${REPOSITORY_URI}"""
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    dockerImage = docker.build "${IMAGE_REPO_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    sh """docker tag ${IMAGE_REPO_NAME}:${IMAGE_TAG} ${REPOSITORY_URI}:${IMAGE_TAG}"""
                    sh """docker push ${REPOSITORY_URI}:${IMAGE_TAG}"""
                }
            }
        }

        stage('Pull Image & Deploy UI Application on EKS Cluster DEV') {
            steps {
                withCredentials([string(credentialsId: 'aws_access_key_id', variable: 'AWS_ACCESS_KEY_ID'),
                                 string(credentialsId: 'aws_secret_access_key', variable: 'AWS_SECRET_ACCESS_KEY')]) {
                    script {
                        dir('kubernetes/') {
                            sh 'aws eks update-kubeconfig --name myAppp-eks-cluster --region ${AWS_DEFAULT_REGION}'
                            sh """aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${REPOSITORY_URI}"""
                            sh 'helm upgrade --install --set image.repository="$REPOSITORY_URI" --set image.tag="${IMAGE_TAG}" myjavaapp myapp/'
                        }
                    }
                }
            }
        }
    }
}
