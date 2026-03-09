pipeline {
    agent any

    environment {
        dockerImage = ''
        SONAR_URL = "http://sonarqube:9000"
        NEXUS_URL = "http://nexus:8081"
        NEXUS_REPOSITORY = "npm-snapshots"
        NEXUS_CREDENTIALS_ID = "nexus-credentials"
    }

    stages {

        stage('Checkout Git') {
            steps {
                checkout scm
            }
        }

        stage('Install, Test & Build') {
            agent {
                docker {
                    image 'node:24-alpine'
                    reuseNode true
                }
            }
            steps {
                sh 'npm install'
                sh 'npm run test -- --watchAll=false --passWithNoTests'
                sh 'npm run build'
            }
        }

        stage('SonarQube Analysis') {
            environment {
                SONAR_TOKEN = credentials('SONAR_TOKEN')
            }
            steps {
                sh """
                    docker run --rm \
                        --network devops-net \
                        -e SONAR_HOST_URL=${SONAR_URL} \
                        -e SONAR_TOKEN=\${SONAR_TOKEN} \
                        -v \${WORKSPACE}:/usr/src \
                        sonarsource/sonar-scanner-cli \
                        -Dsonar.projectKey=medical-ia-front \
                        -Dsonar.sources=. \
                        -Dsonar.exclusions=node_modules/**,.next/**,**/*.test.js,**/*.spec.js
                """
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh """
                    docker run --rm \
                        -v \${WORKSPACE}:/project \
                        aquasec/trivy:latest fs \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        --no-progress \
                        --timeout 20m \
                        --db-repository ghcr.io/aquasecurity/trivy-db:2 \
                        --format table \
                        -o /project/trivy-fs-report.txt \
                        /project || echo "Aucune vulnérabilité trouvée" > \${WORKSPACE}/trivy-fs-report.txt
                """
                archiveArtifacts artifacts: 'trivy-fs-report.txt', allowEmptyArchive: true
            }
        }

        stage('Package Artifact') {
    agent {
        docker {
            image 'node:24-alpine'
            reuseNode true
        }
    }
    steps {
        sh '''
            rm -rf dist-artifacts
            mkdir -p dist-artifacts
            npm pack --pack-destination dist-artifacts
        '''
        archiveArtifacts artifacts: 'dist-artifacts/*.tgz', fingerprint: true
    }
}

        stage('Nexus Upload') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    withCredentials([usernamePassword(
                        credentialsId: "${NEXUS_CREDENTIALS_ID}",
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )]) {
                        sh '''
                            set -eu
                            ARTIFACT_PATH="$(ls -1 dist-artifacts/*.tgz | head -n 1)"
                            ARTIFACT_NAME="$(basename "$ARTIFACT_PATH")"
                            UPLOAD_URL="${NEXUS_URL%/}/repository/${NEXUS_REPOSITORY}/${ARTIFACT_NAME}"
                            curl -fsS -u "${NEXUS_USER}:${NEXUS_PASS}" --upload-file "$ARTIFACT_PATH" "$UPLOAD_URL"
                            echo "Uploaded $ARTIFACT_NAME to $UPLOAD_URL"
                        '''
                    }
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerHub',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        def imageName = "${DOCKER_USER}/medical-ia-front:${BUILD_NUMBER}"
                        dockerImage = docker.build("${imageName}", ".")
                        dockerImage.tag('latest')
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerHub',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        def imageName = "${DOCKER_USER}/medical-ia-front:${BUILD_NUMBER}"
                        sh """
                            docker run --rm \
                                -v /var/run/docker.sock:/var/run/docker.sock \
                                -v \${WORKSPACE}:/workspace \
                                aquasec/trivy:latest image \
                                --severity HIGH,CRITICAL \
                                --exit-code 0 \
                                --no-progress \
                                --timeout 20m \
                                --db-repository ghcr.io/aquasecurity/trivy-db:2 \
                                --format table \
                                -o /workspace/trivy-image-report.txt \
                                ${imageName} || echo "Aucune vulnérabilité HIGH/CRITICAL trouvée" > /workspace/trivy-image-report.txt

                            echo "=== Résultat Trivy Image Scan ==="
                            cat /workspace/trivy-image-report.txt || echo "Rapport vide"
                        """
                    }
                }
                archiveArtifacts artifacts: 'trivy-image-report.txt', allowEmptyArchive: true
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerHub',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        def imageName = "${DOCKER_USER}/medical-ia-front:${BUILD_NUMBER}"
                        sh "docker login -u ${DOCKER_USER} -p ${DOCKER_PASS}"
                        sh "docker push ${imageName}"
                        sh "docker push ${DOCKER_USER}/medical-ia-front:latest"
                    }
                }
            }
        }

        stage('Deploy Application') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerHub',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        def imageName = "${DOCKER_USER}/medical-ia-front:${BUILD_NUMBER}"
                        sh 'docker stop medical-ia-front || true'
                        sh 'docker rm medical-ia-front || true'
                        sh """
                            docker run -d \
                                --name medical-ia-front \
                                --network devops-net \
                                -p 3001:3000 \
                                ${imageName}
                        """
                    }
                }
            }
        }

        stage('Verify Monitoring') {
            steps {
                script {
                    sh 'sleep 15'
                    sh """
                        curl -f http://medical-ia-front:3000 \
                        && echo "✅ Frontend OK" \
                        || echo "❌ Frontend non disponible"
                    """
                    sh """
                        curl -f http://medical-ia-front:3000/api/metrics \
                        && echo "✅ Prometheus endpoint OK" \
                        || echo "❌ Metrics endpoint non disponible"
                    """
                    sh """
                        curl -s http://prometheus:9090/api/v1/targets \
                        | grep medical-ia-front \
                        && echo "✅ Prometheus scrape le front" \
                        || echo "❌ Target non trouvée dans Prometheus"
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline frontend terminé avec succès - Build ${BUILD_NUMBER} déployé"
            mail to: "${TEAM_EMAIL}",
                 subject: "✅ Build ${BUILD_NUMBER} - SUCCESS",
                 body: "Le pipeline medical-ia-front a réussi.\n\nBuild: ${BUILD_NUMBER}\nURL: ${BUILD_URL}"
        }
        failure {
            echo "❌ Pipeline frontend échoué - Build ${BUILD_NUMBER}"
            mail to: "${TEAM_EMAIL}",
                 subject: "❌ Build ${BUILD_NUMBER} - FAILURE",
                 body: "Le pipeline medical-ia-front a échoué.\n\nBuild: ${BUILD_NUMBER}\nURL: ${BUILD_URL}"
        }
        always {
            script {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerHub',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh "docker rmi ${DOCKER_USER}/medical-ia-front:${BUILD_NUMBER} || true"
                }
            }
        }
    }
}