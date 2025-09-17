pipeline {
    agent any
    tools {
        maven 'maven-3.9.11'
    }
    environment {
        SONAR_HOST_URL = "http://sonarqube:9000"
        DOCKER_IMAGE = "spring-petclinic:${BUILD_NUMBER}"
        DOCKER_HUB_REPO = "mahmoudmo123/spring-pipeline"
        APP_PORT = "8086"
    }

    stages {
        stage("Build") {
            steps {
                echo "Cleaning workspace and checking out code"
                deleteDir()
                checkout scm
                echo "Executing build (skip tests)"
                sh "mvn clean install -DskipTests"
            }
        }

        stage("Test & SonarQube") {
            steps {
                echo "Running SonarQube analysis"
                withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_AUTH_TOKEN')]) {
                    sh """
                        mvn test sonar:sonar \
                        -Dsonar.login=$SONAR_AUTH_TOKEN \
                        -Dsonar.projectKey=my-app \
                        -Dsonar.host.url=$SONAR_HOST_URL
                    """
                }
            }
        }

        stage("Push artifacts to Nexus") {
            steps {
                echo "Preparing to upload artifact to Nexus"
                withCredentials([usernamePassword(credentialsId: 'NEXUS_CREDENTIALS', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    script {
                        def version = sh(script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout", returnStdout: true).trim()
                        echo "Project version detected: ${version}"
                        def nexusRepo = version.endsWith("SNAPSHOT") ? "maven-snapshots" : "maven-releases"
                        echo "Deploying to Nexus repository: ${nexusRepo}"

                        def jarFiles = sh(script: "ls target/*.jar || true", returnStdout: true).trim()
                        if (!jarFiles) { error "No jar file found in target/ directory. Build might have failed." }
                        def jarFile = jarFiles.split("\n")[0]

                        echo "Deploying jar: ${jarFile}"
                        nexusArtifactUploader(
                            nexusVersion: 'nexus3',
                            protocol: 'http',
                            nexusUrl: 'nexus:8081',
                            repository: nexusRepo,
                            credentialsId: 'NEXUS_CREDENTIALS',
                            groupId: 'org.springframework.samples',
                            version: version,
                            artifacts: [[artifactId: 'spring-petclinic', classifier: '', file: jarFile, type: 'jar']]
                        )
                    }
                }
            }
        }

        stage("Build Docker Image") {
            steps {
                script {
                    echo "Building Docker image: ${DOCKER_IMAGE}"
                    sh "docker build -t ${DOCKER_IMAGE} ."

                    def dockerHubImage = "${DOCKER_HUB_REPO}:${BUILD_NUMBER}"
                    sh "docker tag ${DOCKER_IMAGE} ${dockerHubImage}"

                    // Docker Hub push
                    withCredentials([usernamePassword(credentialsId: 'DOCKER_HUB_CREDENTIALS', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
                        sh "docker push ${dockerHubImage}"
                    }
                }
            }
        }

        stage("Deploy & Monitoring") {
            steps {
                script {
                    echo "Deploying app + Prometheus + Grafana with Docker Compose"

                    dir("${WORKSPACE}") {
                        // Check prometheus.yml exists and is a file
                        sh """
                            if [ ! -f prometheus.yml ]; then
                                echo '❌ Missing prometheus.yml in workspace root'
                                exit 1
                            fi
                            echo '✅ prometheus.yml found'
                            ls -la prometheus.yml
                        """

                        // Ensure Docker network exists
                        sh "docker network inspect devops-network >/dev/null 2>&1 || docker network create devops-network"

                        // Bring down existing stack first
                        sh "docker compose -f docker-compose.monitoring.yml down || true"

                        // Clean up orphaned containers
                        sh "docker container prune -f || true"

                        // Start the monitoring stack
                        sh """
                            BUILD_NUMBER=${BUILD_NUMBER} \
                            APP_PORT=${APP_PORT} \
                            WORKSPACE=${WORKSPACE} \
                            docker compose -f docker-compose.monitoring.yml up -d --force-recreate
                        """

                        // Wait and check container status
                        sh "sleep 10"
                        sh "docker compose -f docker-compose.monitoring.yml ps"
                    }

                    echo "✅ App running at: http://localhost:${APP_PORT}"
                    echo "✅ Prometheus available at: http://localhost:9090"
                    echo "✅ Grafana available at: http://localhost:3000 (admin/admin)"
                }
            }
        }
    }

    post {
        failure {
            echo "❌ Pipeline failed. Check build logs, credentials, Nexus/DockerHub access, or monitoring configs."
        }
        success {
            echo "✅ Pipeline completed successfully."
            echo "Artifact uploaded to Nexus."
            echo "Docker image built and pushed to Docker Hub."
            echo "Monitoring stack (Prometheus + Grafana) deployed."
        }
    }
}
