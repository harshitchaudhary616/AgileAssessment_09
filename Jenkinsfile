
pipeline {
    agent any

    triggers {
        pollSCM('H/1 * * * *')
    }

    environment {
        IMAGE = 'perkeylynx/university-lms'
    }

    stages {
        stage('Clone Code') {
            steps {
                checkout scm
            }
        }

        stage('Check Tools') {
            steps {
                sh '''
                    export PATH="/Applications/Docker.app/Contents/Resources/bin:/usr/local/bin:/opt/homebrew/bin:$PATH"

                    git --version
                    docker --version
                    kubectl version --client
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    export PATH="/Applications/Docker.app/Contents/Resources/bin:/usr/local/bin:/opt/homebrew/bin:$PATH"
                    docker build -t $IMAGE:$BUILD_NUMBER .
                '''
            }
        }

        stage('Test Application') {
            steps {
                sh '''
                    export PATH="/Applications/Docker.app/Contents/Resources/bin:/usr/local/bin:/opt/homebrew/bin:$PATH"
                    set -e

                    docker rm -f lms-test || true
                    docker run -d --name lms-test \
                        -p 18081:80 $IMAGE:$BUILD_NUMBER

                    sleep 3
                    curl --fail http://localhost:18081/
                    docker rm -f lms-test
                '''
            }
            post {
                always {
                    sh '''
                        export PATH="/Applications/Docker.app/Contents/Resources/bin:/usr/local/bin:/opt/homebrew/bin:$PATH"
                        docker rm -f lms-test || true
                    '''
                }
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    sh '''
                        export PATH="/Applications/Docker.app/Contents/Resources/bin:/usr/local/bin:/opt/homebrew/bin:$PATH"
                        set -e

                        echo "$DH_TOKEN" | docker login \
                            -u "$DH_USER" --password-stdin

                        docker push $IMAGE:$BUILD_NUMBER

                        docker tag $IMAGE:$BUILD_NUMBER $IMAGE:latest
                        docker push $IMAGE:latest

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy and Rollback') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    sh '''
                        export PATH="/Applications/Docker.app/Contents/Resources/bin:/usr/local/bin:/opt/homebrew/bin:$PATH"
                        set -e

                        kubectl apply -f deployment.yaml

                        kubectl set image deployment/university-lms \
                            university-lms=$IMAGE:$BUILD_NUMBER

                        if ! kubectl rollout status \
                            deployment/university-lms --timeout=120s; then

                            echo "Deployment failed. Rolling back..."

                            kubectl rollout undo deployment/university-lms
                            kubectl rollout status \
                                deployment/university-lms --timeout=120s
                            exit 1
                        fi
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'University LMS pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Check the stage logs.'
        }
    }
}
