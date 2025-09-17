pipeline {
    agent any
    tools {
        maven 'maven-3.9.11'
    }
    environment {
        SONAR_HOST_URL = "http://sonarqube:9000"
        DOCKER_IMAGE = "spring-petclinic:${env.BUILD_NUMBER}" 
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
        stage("Test") {
            steps {
                echo "Running SonarQube analysis"
                withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_AUTH_TOKEN')]) {
                    sh "mvn test sonar:sonar -Dsonar.login=$SONAR_AUTH_TOKEN -Dsonar.projectKey=my-app -Dsonar.host.url=$SONAR_HOST_URL"
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
                    // Assuming you have a Dockerfile in your repo
                    sh "docker build -t ${DOCKER_IMAGE} ."

                    // Optional: push to Docker registry
                    // withCredentials([usernamePassword(credentialsId: 'DOCKER_HUB', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    //     sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"
                    //     sh "docker push ${DOCKER_IMAGE}"
                    // }
                }
            }
        }
    }
    post {
        failure {
            echo "Pipeline failed. Check build, credentials, repository permissions, and Maven configuration."
        }
        success {
            echo "Pipeline completed successfully. Artifact should now appear in Nexus and Docker image built."
        }
    }
}
