pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        APP_NAME = "reddit-clone-pipeline"
        RELEASE = "1.0.0"
        DOCKER_USER = "pdahikar"
        DOCKER_PASS = credentials('docker-hub-credentials') // Use Jenkins credentials for security
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/pdahika/a-reddit-clone.git'
            }
        }
        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh '''${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=redit-clone-ci \
                        -Dsonar.projectKey=redit-clone-ci'''
                }
            }
        }
        stage("Quality Gate") {
            steps {
                script {
                    def qg = waitForQualityGate()
                    if (qg.status != 'OK') {
                        error "Pipeline aborted due to quality gate failure: ${qg.status}"
                    }
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('Trivy FS Scan') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Build & Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('', 'docker-hub-credentials') {
                        def docker_image = docker.build("${IMAGE_NAME}")
                        docker_image.push("${RELEASE}-${env.BUILD_NUMBER}")
                        docker_image.push('latest')
                    }
                }
            }
        }
        stage("Trivy Image Scan") {
            steps {
                sh '''
                docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                aquasec/trivy image ${IMAGE_NAME}:latest \
                --no-progress --scanners vuln --exit-code 0 \
                --severity HIGH,CRITICAL --format table > trivyimage.txt
                '''
            }
        }
        stage('Cleanup Artifacts') {
            steps {
                sh '''
                docker rmi ${IMAGE_NAME}:${RELEASE}-${env.BUILD_NUMBER} || true
                docker rmi ${IMAGE_NAME}:latest || true
                '''
            }
        }
        stage("Trigger CD Pipeline") {
            steps {
                script {
                    sh '''
                    curl -v -k --user clouduser:${JENKINS_API_TOKEN} \
                    -X POST -H 'cache-control: no-cache' \
                    -H 'content-type: application/x-www-form-urlencoded' \
                    --data "IMAGE_TAG=${RELEASE}-${env.BUILD_NUMBER}" \
                    'http://ec2-65-2-187-142.ap-south-1.compute.amazonaws.com:8080/job/Reddit-Clone-CD/buildWithParameters?token=gitops-token'
                    '''
                }
            }
        }
    }
}
