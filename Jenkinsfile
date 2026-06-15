pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
    }

    environment {
        IMAGE_NAME = 'spring-petclinic'
        IMAGE_TAG = "${BUILD_NUMBER}"
        MAVEN_SETTINGS = '.ci/maven-settings.xml'
        SMOKE_CONTAINER = 'spring-petclinic-smoke-test'
    }

    stages {
        stage('Checkout') {
            steps {
                deleteDir()
                checkout scm
            }
        }

        stage('Prepare') {
            steps {
                sh '''
                    chmod +x ./mvnw
                    rm -f spring-petclinic-docker-image.tar spring-petclinic-docker-image.tar.gz
                    rm -rf target/docker-image
                '''
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
                sh 'docker build --platform linux/amd64 -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest .'
            }
        }

        stage('Smoke Test Docker Image') {
            steps {
                sh '''
                    docker rm -f ${SMOKE_CONTAINER} 2>/dev/null || true

                    docker run -d --name ${SMOKE_CONTAINER} ${IMAGE_NAME}:latest

                    echo "Waiting for Spring PetClinic container to become available..."

                    for i in $(seq 1 30); do
                        if docker run --rm --network container:${SMOKE_CONTAINER} curlimages/curl:8.10.1 -fsS http://localhost:8080 > /dev/null; then
                            echo "Smoke test passed. Application is reachable on port 8080."
                            docker rm -f ${SMOKE_CONTAINER}
                            exit 0
                        fi

                        echo "Application not ready yet. Retrying..."
                        sleep 2
                    done

                    echo "Smoke test failed. Container logs:"
                    docker logs ${SMOKE_CONTAINER}
                    docker rm -f ${SMOKE_CONTAINER}
                    exit 1
                '''
            }
        }

        stage('Export Docker Image') {
            steps {
                sh '''
                    mkdir -p target/docker-image
                    rm -f target/docker-image/*.tar target/docker-image/*.tar.gz

                    docker save ${IMAGE_NAME}:latest -o target/docker-image/${IMAGE_NAME}-${IMAGE_TAG}.tar
                    gzip target/docker-image/${IMAGE_NAME}-${IMAGE_TAG}.tar
                '''

                archiveArtifacts artifacts: 'target/docker-image/*.tar.gz', fingerprint: true
            }
        }
    }

    post {
        always {
            sh 'docker rm -f ${SMOKE_CONTAINER} 2>/dev/null || true'
        }

        success {
            echo "Pipeline completed successfully. Built and exported runnable Docker image: ${IMAGE_NAME}:${IMAGE_TAG}"
        }

        failure {
            echo 'Pipeline failed.'
        }
    }
}
