pipeline {
    agent any
    tools {
        maven 'maven-3.9.11'
    }
    environment {
        SONAR_HOST_URL = "http://sonarqube:9000"
        DOCKER_IMAGE = "spring-petclinic:${env.BUILD_NUMBER}"
        DOCKER_HUB_REPO = "mahmoudmo123/spring-pipeline"
        APP_CONTAINER = "spring-petclinic-app"
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

        stage("Build & Push Docker Image") {
            steps {
                script {
                    echo "Building Docker image: ${DOCKER_IMAGE}"
                    sh "docker build -t ${DOCKER_IMAGE} ."

                    def dockerHubImage = "${DOCKER_HUB_REPO}:${env.BUILD_NUMBER}"
                    sh "docker tag ${DOCKER_IMAGE} ${dockerHubImage}"
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
                    echo "Deploying app container for Prometheus monitoring"
                    sh """
                        docker rm -f ${APP_CONTAINER} || true
                        docker run -d --name ${APP_CONTAINER} \
                          --network devops-network \
                          -p ${APP_PORT}:8080 \
                          ${DOCKER_HUB_REPO}:${env.BUILD_NUMBER}
                    """
                    echo "App deployed on http://localhost:${APP_PORT}"
                    echo "Prometheus will scrape metrics from /actuator/prometheus"
                }
            }
        }
    }

    post {
        failure {
            echo "Pipeline failed. Check build, credentials, repository permissions, and Maven configuration."
        }
        success {
            echo "Pipeline completed successfully. Artifact is in Nexus, Docker image pushed, and container deployed for monitoring."
        }
    }
}
