pipeline {
    agent any

    environment {
        IMAGE_NAME = 'spring-petclinic'
        IMAGE_TAG = "${BUILD_NUMBER}"
        MAVEN_SETTINGS = '.ci/maven-settings.xml'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Prepare') {
            steps {
                sh 'chmod +x ./mvnw'
            }
        }

        stage('Compile') {
            steps {
                sh './mvnw -B -s ${MAVEN_SETTINGS} -DskipTests compile'
            }
        }

        stage('Test') {
            steps {
                sh './mvnw -B -s ${MAVEN_SETTINGS} test'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Package') {
            steps {
                sh './mvnw -B -s ${MAVEN_SETTINGS} -DskipTests package'
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest .'
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully. Built Docker image: ${IMAGE_NAME}:${IMAGE_TAG}"
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}
