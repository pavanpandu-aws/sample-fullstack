pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = '634441478571.dkr.ecr.us-west-1.amazonaws.com'
        DOCKER_LATEST_TAG = 'latest'
        VERSION = '1.0'
    }

    parameters {
        choice(name: 'BRANCH_NAME', choices: ['dev', 'test', 'prod'], description: 'Branch name to build')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Set Environment') {
            steps {
                script {
                    if (env.BRANCH_NAME.startsWith('dev')) {
                        env.BACKEND_IMAGE_TAG = "dev-backend-${env.BUILD_NUMBER}-${env.GIT_COMMIT.substring(0, 7)}-${env.VERSION}"
                        env.FRONTEND_IMAGE_TAG = "dev-frontend-${env.BUILD_NUMBER}-${env.GIT_COMMIT.substring(0, 7)}-${env.VERSION}"
                    } else if (env.BRANCH_NAME.startsWith('test')) {
                        env.BACKEND_IMAGE_TAG = "test-backend-${env.BUILD_NUMBER}-${env.GIT_COMMIT.substring(0, 7)}-${env.VERSION}"
                        env.FRONTEND_IMAGE_TAG = "test-frontend-${env.BUILD_NUMBER}-${env.GIT_COMMIT.substring(0, 7)}-${env.VERSION}"
                    } else if (env.BRANCH_NAME.startsWith('prod')) {
                        env.BACKEND_IMAGE_TAG = "prod-backend-${env.BUILD_NUMBER}-${env.GIT_COMMIT.substring(0, 7)}-${env.VERSION}"
                        env.FRONTEND_IMAGE_TAG = "prod-frontend-${env.BUILD_NUMBER}-${env.GIT_COMMIT.substring(0, 7)}-${env.VERSION}"
                    } else {
                        error("Unsupported branch name: ${env.BRANCH_NAME}")
                    }
                }
            }
        }

        stage('prepare') {
            steps {
                sh "ansible-vault decrypt --vault-id /tmp/vault_id demo.pem"
                sh "chmod 400 demo.pem"
                sh "ansible-playbook -i inventory install_tools.yml"
            }
        }
        
        stage('Frontend Unit Test') {
            environment {
                NODE_ENV = 'test'
            }
            steps {
                dir('frontend') {
                    sh 'npm install'
                    sh 'npm run test'
                }
            }
            post {
                always {
                    junit "frontend/reports/${env.FRONTEND_IMAGE_TAG}/**/*.xml"
                }
            }
        }

        stage('Backend Unit Test') {
            environment {
                MAVEN_OPTS = '-Denv=test'
            }
            steps {
                dir('backend') {
                    sh 'mvn test'
                }
            }
            post {
                always {
                    junit "backend/target/surefire-reports/${env.BACKEND_IMAGE_TAG}/**/*.xml"
                }
            }
        }

        stage('Frontend Sonar Scanning') {
            environment {
                NODE_ENV = 'test'
            }
            steps {
                withSonarQubeEnv('SonarQube') {
                    dir('frontend') {
                        sh 'npm run sonar'
                    }
                }
            }
        }

        stage('Backend Sonar Scanning') {
            environment {
                MAVEN_OPTS = '-Denv=test'
            }
            steps {
                withSonarQubeEnv('SonarQube') {
                    dir('backend') {
                        sh 'mvn sonar:sonar'
                    }
                }
            }
        }

        stage('Frontend Code Analysis with Checkstyle') {
            environment {
                NODE_ENV = 'test'
            }
            steps {
                dir('frontend') {
                    sh 'npm run lint'
                }
            }
            post {
                always {
                    checkstyle "frontend/reports/checkstyle-${env.FRONTEND_IMAGE_TAG}.xml"
                }
            }
        }

        stage('Backend Code Analysis with Checkstyle') {
            environment {
                MAVEN_OPTS = '-Denv=test'
            }
            steps {
                dir('backend') {
                    sh 'mvn checkstyle:checkstyle'
                }
            }
            post {
                always {
                    checkstyle "backend/target/checkstyle-${env.BACKEND_IMAGE_TAG}.xml"
                }
            }
        }
        
        stage('Build Backend Image') {
            steps {
                dir('backend') {
                    sh 'mvn clean install'
                    sh "docker build -t ${env.BACKEND_IMAGE_TAG} ."
                }
            }
        }

        stage('Build Frontend Image') {
            steps {
                dir('frontend') {
                    sh "docker build -t ${env.FRONTEND_IMAGE_TAG} ."
                }
            }
        }

        stage('Push Images to ECR') {
            environment {
                AWS_REGION = 'us-east-1'
            }
            steps {
                script {
                    withCredentials([string(credentialsId: 'aws-credentials', variable: 'AWS_CREDENTIALS')]) {
                        sh "aws ecr get-login-password --region ${env.AWS_REGION} | docker login --username AWS --password-stdin ${env.DOCKER_REGISTRY}"
                        sh "docker tag ${env.BACKEND_IMAGE_TAG} ${DOCKER_REGISTRY}/backend:${env.BACKEND_IMAGE_TAG}"
                        sh "docker tag ${env.BACKEND_IMAGE_TAG} ${DOCKER_REGISTRY}/backend:${DOCKER_LATEST_TAG}"
                        sh "docker tag ${env.FRONTEND_IMAGE_TAG} ${DOCKER_REGISTRY}/frontend:${env.FRONTEND_IMAGE_TAG}"
                        sh "docker tag ${env.FRONTEND_IMAGE_TAG} ${DOCKER_REGISTRY}/frontend:${DOCKER_LATEST_TAG}"
                        
                        sh "docker push ${DOCKER_REGISTRY}/backend:${env.BACKEND_IMAGE_TAG}"
                        sh "docker push ${DOCKER_REGISTRY}/backend:${DOCKER_LATEST_TAG}"
                        sh "docker push ${DOCKER_REGISTRY}/frontend:${env.FRONTEND_IMAGE_TAG}"
                        sh "docker push ${DOCKER_REGISTRY}/frontend:${DOCKER_LATEST_TAG}"
                    }
                }
            }
        }
    }
}
