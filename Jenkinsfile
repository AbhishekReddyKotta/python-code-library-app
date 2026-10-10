pipeline {
    agent {
        node {
            label 'dev'
        }
    }
    stages {
        stage ('code') {
            steps {
                git 'https://github.com/AbhishekReddyKotta/python-code-library-app.git'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                script {
                    // def scannerHome = tool 'ark-sonar';
                    withSonarQubeEnv('ark-sonar') {
                        // sh "${scannerHome}/bin/sonar-scanner"
                        sh '''
                            sonar-scanner \
                                -Dsonar.projectKey=ark-sonar \
                                -Dsonar.sources=.
                        '''
                    }
                }
            }
        }
        stage('Quality Gates') {
            steps {
                script {
                    waitForQualityGate abortPipeline: true, credentialsId: 'sonar'
                }
            }
        }
        stage ('Image Build') {
            steps {
                sh 'docker build -t appimage:v${BUILD_NUMBER} .'
                sh 'docker build -t dbimage:v${BUILD_NUMBER} database'
                sh 'docker build -t borrowimage:v${BUILD_NUMBER} borrow'
                sh 'docker build -t authimage:v${BUILD_NUMBER} auth'
                sh 'docker build -t bookimage:v${BUILD_NUMBER} book'
            }
        }
        stage ('Image Scan') {
            steps {
                sh ' trivy image appimage:v${BUILD_NUMBER} >> appimage-trivy-report.txt'
                sh ' trivy image dbimage:v${BUILD_NUMBER} >> dbimage-trivy-report.txt'
                sh ' trivy image borrowimage:v${BUILD_NUMBER} >> borrowimage-trivy-report.txt'
                sh ' trivy image authimage:v${BUILD_NUMBER} >> authimage-trivy-report.txt'
                sh ' trivy image bookimage:v${BUILD_NUMBER} >> bookimage-trivy-report.txt'
            }
        }
        stage ('Image Registry') {
            steps {
                script {
                    // This step should not normally be used in your script. Consult the inline help for details.
                    withDockerRegistry(credentialsId: 'dockerhub') {
                        // some block
                        sh 'docker tag appimage:v${BUILD_NUMBER} arkotta27/python-code-library-app:v${BUILD_NUMBER}'
                        sh 'docker tag dbimage:v${BUILD_NUMBER} arkotta27/python-code-library-db:v${BUILD_NUMBER}'
                        sh 'docker tag borrowimage:v${BUILD_NUMBER} arkotta27/python-code-library-borrow:v${BUILD_NUMBER}'
                        sh 'docker tag authimage:v${BUILD_NUMBER} arkotta27/python-code-library-auth:v${BUILD_NUMBER}'
                        sh 'docker tag bookimage:v${BUILD_NUMBER} arkotta27/python-code-library-book:v${BUILD_NUMBER}'
                        sh 'docker push arkotta27/python-code-library-app:v${BUILD_NUMBER}'
                        sh 'docker push arkotta27/python-code-library-db:v${BUILD_NUMBER}'
                        sh 'docker push arkotta27/python-code-library-borrow:v${BUILD_NUMBER}'
                        sh 'docker push arkotta27/python-code-library-auth:v${BUILD_NUMBER}'
                        sh 'docker push arkotta27/python-code-library-book:v${BUILD_NUMBER}'
                    }
                }
            }
        }
        stage('Deploy') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'swarm_id',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    ),
                    usernamePassword(
                        credentialsId: 'dockerhub_id',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        set -eu

                        SWARM_HOST="172.31.3.35"
                        export DOCKER_CONFIG="$(mktemp -d)"
                        chmod 700 "$DOCKER_CONFIG"

                        cleanup() {
                            docker logout >/dev/null 2>&1 || true
                            rm -rf "$DOCKER_CONFIG"
                        }
                        trap cleanup EXIT

                        printf '%s' "$DOCKERHUB_TOKEN" |
                            docker login --username "$DOCKERHUB_USER" \
                            --password-stdin

                        sed "s/\\${BUILD_NUMBER}/${BUILD_NUMBER}/g" \
                            compose.yml > deploy-compose.yml

                        scp -i "$SSH_KEY" \
                            -o BatchMode=yes \
                            -o StrictHostKeyChecking=yes \
                            deploy-compose.yml \
                            "$SSH_USER@$SWARM_HOST:/tmp/deploy-compose.yml"

                        ssh -i "$SSH_KEY" \
                            -o BatchMode=yes \
                            -o StrictHostKeyChecking=yes \
                            "$SSH_USER@$SWARM_HOST" \
                            "sudo docker stack deploy \
                            --with-registry-auth \
                            -c /tmp/deploy-compose.yml \
                            myapp"

                        ssh -i "$SSH_KEY" \
                            -o BatchMode=yes \
                            -o StrictHostKeyChecking=yes \
                            "$SSH_USER@$SWARM_HOST" \
                            "sudo docker stack services myapp"
                    '''
                }
            }
        }
    }
}
