pipeline {
    agent any

    tools {
        maven 'maven3.9.16'
        jdk 'java17'
    }

    options {
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
    }

    environment {
        APP_NAME         = 'country-chicken-backend'
        NEXUS_MAVEN_URL  = '54.145.178.156:8081'
        NEXUS_DOCKER_URL = '54.145.178.156:8082'
        MAVEN_REPO       = 'maven-releases'
        DOCKER_REPO      = 'docker-releases'
        GROUP_ID         = 'com.countrychicken'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Set Version') {
            steps {
                script {
                    env.VERSION = sh(
                        script: 'mvn help:evaluate -Dexpression=project.version -q -DforceStdout',
                        returnStdout: true
                    ).trim()

                    if (!(env.VERSION ==~ /1\.0\.\d+/)) {
                        error 'Expected release version: 1.0.0, 1.0.1, ...'
                    }

                    env.JAR_NAME = "${env.APP_NAME}-${env.VERSION}.jar"
                    echo "Release version: ${env.VERSION}"
                }
            }
        }

        stage('Build JAR') {
            steps {
                sh 'mvn -B clean package -DskipTests'
            }
        }

        stage('Upload JAR to Nexus') {
            steps {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: env.NEXUS_MAVEN_URL,
                    groupId: env.GROUP_ID,
                    version: env.VERSION,
                    repository: env.MAVEN_REPO,
                    credentialsId: 'nexus-credentials',
                    artifacts: [
                        [
                            artifactId: env.APP_NAME,
                            classifier: '',
                            file: "target/${env.JAR_NAME}",
                            type: 'jar'
                        ]
                    ]
                )
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                      -t "$NEXUS_DOCKER_URL/$DOCKER_REPO/$APP_NAME:$VERSION" .
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        set +x
                        export DOCKER_CONFIG="$(mktemp -d)"
                        trap 'rm -rf "$DOCKER_CONFIG"' EXIT

                        printf '%s' "$DOCKER_PASS" |
                          docker login "$NEXUS_DOCKER_URL" \
                            -u "$DOCKER_USER" --password-stdin

                        docker push \
                          "$NEXUS_DOCKER_URL/$DOCKER_REPO/$APP_NAME:$VERSION"

                        docker logout "$NEXUS_DOCKER_URL"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'JAR and Docker image published successfully'
        }
        failure {
            echo 'CI pipeline failed. Check the failed stage logs.'
        }
        always {
            cleanWs()
        }
    }
}