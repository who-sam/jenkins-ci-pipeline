pipeline {
    agent any
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        FRONTEND_IMAGE = 'whosam1/notes-app-frontend'
        BACKEND_IMAGE = 'whosam1/notes-app-backend'
        GITHUB_TOKEN = credentials('github-token')
        // Use commit hash for immutable tags
        IMAGE_TAG = "${GIT_COMMIT}"
    }
    stages {
        stage('Checkout Notes App Code') {
            steps {
                git branch: 'main', url: 'https://github.com/who-sam/MIND.git'
            }
        }

        stage('Checkout Manifests Repo') {
            steps {
                dir('manifests-repo') {
                    git branch: 'main', url: 'https://github.com/who-sam/argocd-pipeline.git'
                }
            }
        }

        stage('Build Docker Images') {
            parallel {
                stage('Build Frontend') {
                    steps {
                        script {
                            sh """
                            docker build -t $FRONTEND_IMAGE:$IMAGE_TAG -f frontend/Dockerfile ./frontend
                            docker tag $FRONTEND_IMAGE:$IMAGE_TAG $FRONTEND_IMAGE:latest
                            """
                        }
                    }
                }
                stage('Build Backend') {
                    steps {
                        script {
                            sh """
                            docker build -t $BACKEND_IMAGE:$IMAGE_TAG -f backend/Dockerfile ./backend
                            docker tag $BACKEND_IMAGE:$IMAGE_TAG $BACKEND_IMAGE:latest
                            """
                        }
                    }
                }
            }
        }

        stage('Run Tests') {
            parallel {
                stage('Test Frontend') {
                    steps {
                        script {
                            // Add your frontend tests here
                            echo "Running frontend tests..."
                        }
                    }
                }
                stage('Test Backend') {
                    steps {
                        script {
                            // Uncomment and modify your backend tests
                            // dir('backend') {
                            //     sh 'go test ./...'
                            // }
                            echo "Running backend tests..."
                        }
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            parallel {
                stage('Push Frontend') {
                    steps {
                        script {
                            sh """
                            docker login -u $DOCKERHUB_CREDENTIALS_USR -p $DOCKERHUB_CREDENTIALS_PSW
                            docker push $FRONTEND_IMAGE:$IMAGE_TAG
                            docker push $FRONTEND_IMAGE:latest
                            """
                        }
                    }
                }
                stage('Push Backend') {
                    steps {
                        script {
                            sh """
                            docker push $BACKEND_IMAGE:$IMAGE_TAG
                            docker push $BACKEND_IMAGE:latest
                            """
                        }
                    }
                }
            }
        }

        stage('Update K8s Manifests') {
            steps {
                script {
                    // Fix: Update the correct path (remove ArgoCD-Pipeline subdirectory)
                    dir('manifests-repo') {
                        sh """
                        # Update frontend deployment with new image
                        sed -i 's|image: whosam1/notes-app-frontend:.*|image: $FRONTEND_IMAGE:$IMAGE_TAG|' frontend-deployment.yaml

                        # Update backend deployment with new image
                        sed -i 's|image: whosam1/notes-app-backend:.*|image: $BACKEND_IMAGE:$IMAGE_TAG|' backend-deployment.yaml
                        """

                        // Verify the changes
                        sh '''
                        echo "=== Updated Frontend Deployment ==="
                        cat frontend-deployment.yaml | grep image:
                        echo "=== Updated Backend Deployment ==="
                        cat backend-deployment.yaml | grep image:
                        '''
                    }
                }
            }
        }

        stage('Commit and Push Manifests') {
            steps {
                script {
                    dir('manifests-repo') {
                        sh """
                        git config user.name "jenkins"
                        git config user.email "jenkins@example.com"
                        git add frontend-deployment.yaml backend-deployment.yaml
                        git status
                        git commit -m "CI: Update Notes App image tags to ${IMAGE_TAG}"
                        git push https://${GITHUB_TOKEN}@github.com/who-sam/argocd-pipeline.git main
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                // Clean up Docker images to free up space
                sh """
                docker rmi $FRONTEND_IMAGE:$IMAGE_TAG || true
                docker rmi $FRONTEND_IMAGE:latest || true
                docker rmi $BACKEND_IMAGE:$IMAGE_TAG || true
                docker rmi $BACKEND_IMAGE:latest || true
                """
            }
            cleanWs()
        }
        success {
            echo "Pipeline completed successfully! ArgoCD should now sync the new images."
        }
        failure {
            echo "Pipeline failed! Check the logs for details."
            // You can add notification steps here (email, Slack, etc.)
        }
    }
}
