pipeline {
    agent any
    
    triggers {

      githubPush()

    }

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        FRONTEND_IMAGE = 'whosam1/notes-app-frontend'
        BACKEND_IMAGE = 'whosam1/notes-app-backend'
        GITHUB_TOKEN = credentials('github-token')
        IMAGE_TAG = "${env.GIT_COMMIT.take(7)}"
        BUILD_DATE = new Date().format('yyyyMMdd-HHmmss')
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
                    git branch: 'main', 
                        credentialsId: 'github-token',
                        url: 'https://github.com/who-sam/argocd-pipeline.git'
                }
            }
        }

        stage('Build Docker Images') {
            parallel {
                stage('Build Frontend') {
                    steps {
                        script {
                            sh """
                            docker build \
                                -t ${FRONTEND_IMAGE}:${IMAGE_TAG} \
                                -t ${FRONTEND_IMAGE}:latest \
                                --build-arg BUILD_DATE=${BUILD_DATE} \
                                -f frontend/Dockerfile ./frontend
                            """
                        }
                    }
                }
                stage('Build Backend') {
                    steps {
                        script {
                            sh """
                            docker build \
                                -t ${BACKEND_IMAGE}:${IMAGE_TAG} \
                                -t ${BACKEND_IMAGE}:latest \
                                --build-arg BUILD_DATE=${BUILD_DATE} \
                                -f backend/Dockerfile ./backend
                            """
                        }
                    }
                }
            }
        }

        stage('Security Scan') {
            parallel {
                stage('Scan Frontend Image') {
                    steps {
                        script {
                            sh """
                            echo "Running Trivy scan on frontend image..."
                            docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                                aquasec/trivy:latest image --severity HIGH,CRITICAL \
                                ${FRONTEND_IMAGE}:${IMAGE_TAG} || true
                            """
                        }
                    }
                }
                stage('Scan Backend Image') {
                    steps {
                        script {
                            sh """
                            echo "Running Trivy scan on backend image..."
                            docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                                aquasec/trivy:latest image --severity HIGH,CRITICAL \
                                ${BACKEND_IMAGE}:${IMAGE_TAG} || true
                            """
                        }
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    sh """
                    echo \$DOCKERHUB_CREDENTIALS_PSW | docker login -u \$DOCKERHUB_CREDENTIALS_USR --password-stdin
                    
                    # Push frontend images
                    docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                    docker push ${FRONTEND_IMAGE}:latest
                    
                    # Push backend images
                    docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
                    docker push ${BACKEND_IMAGE}:latest
                    
                    docker logout
                    """
                }
            }
        }

        stage('Update K8s Manifests') {
            steps {
                script {
                    dir('manifests-repo') {
                        sh """
                        # Update frontend deployment with new image
                        sed -i 's|image: whosam1/notes-app-frontend:.*|image: ${FRONTEND_IMAGE}:${IMAGE_TAG}|' frontend-deployment.yaml

                        # Update backend deployment with new image
                        sed -i 's|image: whosam1/notes-app-backend:.*|image: ${BACKEND_IMAGE}:${IMAGE_TAG}|' backend-deployment.yaml
                        
                        # Verify the changes
                        echo "=== Updated Frontend Deployment ==="
                        grep "image:" frontend-deployment.yaml
                        echo "=== Updated Backend Deployment ==="
                        grep "image:" backend-deployment.yaml
                        """
                    }
                }
            }
        }

        stage('Commit and Push Manifests') {
            steps {
                script {
                    dir('manifests-repo') {
                        sh """
                        git config user.name "Jenkins CI"
                        git config user.email "jenkins@ci.example.com"
                        git add frontend-deployment.yaml backend-deployment.yaml
                        
                        if git diff --staged --quiet; then
                            echo "No changes to commit"
                        else
                            git commit -m "CI: Update Notes App images to ${IMAGE_TAG}
                            
                            Frontend: ${FRONTEND_IMAGE}:${IMAGE_TAG}
                            Backend: ${BACKEND_IMAGE}:${IMAGE_TAG}
                            Build: ${BUILD_DATE}
                            Commit: ${env.GIT_COMMIT}"
                            
                            git push https://${GITHUB_TOKEN}@github.com/who-sam/argocd-pipeline.git main
                        fi
                        """
                    }
                }
            }
        }

        stage('Trigger ArgoCD Sync') {
            steps {
                script {
                    sh """
                    echo "ArgoCD will automatically sync the changes"
                    echo "Application: notes-app"
                    echo "Image Tags: ${IMAGE_TAG}"
                    """
                }
            }
        }
    }

    post {
        always {
            script {
                // Clean up Docker images to free up space
                sh """
                docker rmi ${FRONTEND_IMAGE}:${IMAGE_TAG} || true
                docker rmi ${FRONTEND_IMAGE}:latest || true
                docker rmi ${BACKEND_IMAGE}:${IMAGE_TAG} || true
                docker rmi ${BACKEND_IMAGE}:latest || true
                docker system prune -f || true
                """
                // Clean workspace
                deleteDir()
            }
        }
        success {
            echo """
            ================================================
            Pipeline completed successfully!
            ================================================
            Frontend Image: ${FRONTEND_IMAGE}:${IMAGE_TAG}
            Backend Image: ${BACKEND_IMAGE}:${IMAGE_TAG}
            Build Date: ${BUILD_DATE}
            
            ArgoCD should sync automatically.
            Monitor deployment at: ArgoCD Dashboard
            ================================================
            """
        }
        failure {
            echo """
            ================================================
            Pipeline failed!
            ================================================
            Check the logs above for details.
            Stage: ${env.STAGE_NAME}
            ================================================
            """
        }
    }
}
