pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIAL_ID = 'mlops-jenkins-dockerhub-token'
        DOCKERHUB_REGISTRY = 'https://registry.hub.docker.com'
        DOCKERHUB_REPOSITORY = 'skmohan7984/mlops-jenkins-01'
    }

    stages {

        stage('Lint Code') {
            steps {
                script {
                    echo 'Linting Python Code...'
                    sh '''
                        python3 -m venv venv
                        . venv/bin/activate
                        pip install --upgrade pip
                        pip install -r requirements.txt
                        pylint app.py train.py --exit-zero --output=pylint-report.txt
                        flake8 app.py train.py --ignore=E501,E302 --output-file=flake8-report.txt || true
                        black app.py train.py || true
                    '''
                }
            }
        }

        stage('Test Code') {
            steps {
                script {
                    echo 'Running Tests...'
                    sh '''
                        . venv/bin/activate
                        pytest tests/ || true
                    '''
                }
            }
        }

        stage('Trivy FS Scan') {
            steps {
                script {
                    echo 'Scanning filesystem with Trivy...'
                    sh 'trivy fs . --format table -o trivy-fs-report.txt || true'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo 'Building Docker Image...'
                    dockerImage = docker.build("${DOCKERHUB_REPOSITORY}:latest")
                }
            }
        }

        stage('Trivy Docker Image Scan') {
            steps {
                script {
                    echo 'Scanning Docker Image with Trivy...'
                    sh "trivy image ${DOCKERHUB_REPOSITORY}:latest --format table -o trivy-image-report.txt || true"
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    echo 'Pushing Docker Image...'
                    docker.withRegistry("${DOCKERHUB_REGISTRY}", "${DOCKERHUB_CREDENTIAL_ID}") {
                        dockerImage.push("latest")
                    }
                }
            }
        }

        stage('Deploy to ECS') {
            steps {
                script {
                    echo 'Deploying to Amazon ECS...'
                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'mlops-jenkins-dockerhub-token'
                    ]]) {
                        sh '''
                            aws ecs update-service \
                              --cluster mlops-jenkins-ecs-1 \
                              --service mlops-jenkins-01-task-definition-service-radwjwae \
                              --force-new-deployment \
                              --region ap-south-1
                        '''
                    }
                }
            }
        }
    }
}
