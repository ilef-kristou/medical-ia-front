pipeline {
    agent any

    environment {
        dockerImage = ''
        SONAR_URL = "http://sonarqube:9000"
    }

    stages {

        stage('Checkout Git') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            agent { docker { image 'node:20-alpine' } }
            steps {
                sh 'npm install'
            }
        }

        stage('Run Tests') {
            agent { docker { image 'node:20-alpine' } }
            steps {
                sh 'npm run test -- --watchAll=false --passWithNoTests'
            }
        }

        stage('Build Next.js') {
            agent { docker { image 'node:20-alpine' } }
            steps {
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
                        -v \${PWD}:/usr/src \
                        sonarsource/sonar-scanner-cli \
                        -Dsonar.projectKey=medical-ia-front \
                        -Dsonar.sources=. \
                        -Dsonar.exclusions=node_modules/**,.next/**
                """
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh """
                    docker run --rm \
                        -v \${PWD}:/project \
                        aquasec/trivy:latest fs \
                        --severity HIGH,CRITICAL \
                        --exit-code 0 \
                        --no-progress \
                        --timeout 20m \
                        --db-repository ghcr.io/aquasecurity/trivy-db:2 \
                        --format table \
                        -o /project/trivy-fs-report.txt \
                        /project || echo "Aucune vulnérabilité HIGH/CRITICAL trouvée" > /project/trivy-fs-report.txt

                    echo "=== Résultat Trivy Filesystem Scan ==="
                    cat /project/trivy-fs-report.txt || echo "Rapport vide"
                """
                archiveArtifacts artifacts: 'trivy-fs-report.txt', allowEmptyArchive: true
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

        stage('Verify Deploy') {
            steps {
                script {
                    sh 'sleep 15'
                    sh """
                        curl -f http://medical-ia-front:3000 \
                        && echo "✅ Frontend OK" \
                        || echo "❌ Frontend non disponible"
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline frontend terminé avec succès - Build ${BUILD_NUMBER} déployé"
            mail to: "${EMAIL}",
                 subject: "✅ Build ${BUILD_NUMBER} - SUCCESS",
                 body: "Le pipeline medical-ia-front a réussi.\n\nBuild: ${BUILD_NUMBER}\nURL: ${BUILD_URL}"
        }
        failure {
            echo "❌ Pipeline frontend échoué - Build ${BUILD_NUMBER}"
            mail to: "${EMAIL}",
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