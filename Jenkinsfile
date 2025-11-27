pipeline {
    agent any
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        FRONTEND_IMAGE = 'whosam1/notes-app-frontend'
        BACKEND_IMAGE = 'whosam1/notes-app-backend'
        GITHUB_TOKEN = credentials('github-token')
    }
    stages {
        stage('Checkout Notes App Code') {
            steps {
                // Clone the Notes App repository
                git branch: 'main', url: 'https://github.com/who-sam/MIND.git'
            }
        }

        stage('Checkout Manifests Repo') {
            steps {
                // Clone the helloapp repo for manifests (in a different directory)
                dir('manifests-repo') {
                    git branch: 'main', url: 'https://github.com/who-sam/argocd-pipeline.git'
                }
            }
        }

        stage('Test Backend') {
            steps {
                dir('backend') {
                    sh 'go test ./...'
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    // Build frontend
                    sh """
                    docker build -t $FRONTEND_IMAGE:${GIT_COMMIT} -f frontend/Dockerfile ./frontend
                    docker tag $FRONTEND_IMAGE:${GIT_COMMIT} $FRONTEND_IMAGE:latest
                    """

                    // Build backend
                    sh """
                    docker build -t $BACKEND_IMAGE:${GIT_COMMIT} -f backend/Dockerfile ./backend
                    docker tag $BACKEND_IMAGE:${GIT_COMMIT} $BACKEND_IMAGE:latest
                    """
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    sh """
                    docker login -u $DOCKERHUB_CREDENTIALS_USR -p $DOCKERHUB_CREDENTIALS_PSW
                    docker push $FRONTEND_IMAGE:${GIT_COMMIT}
                    docker push $FRONTEND_IMAGE:latest
                    docker push $BACKEND_IMAGE:${GIT_COMMIT}
                    docker push $BACKEND_IMAGE:latest
                    """
                }
            }
        }

        stage('Update K8s Manifests') {
            steps {
                script {
                    sh """
                    # Update frontend deployment with new image
                    sed -i 's|image: .*/notes-app-frontend:.*|image: $FRONTEND_IMAGE:${GIT_COMMIT}|' manifests-repo/ArgoCD-Pipeline/frontend-deployment.yaml

                    # Update backend deployment with new image
                    sed -i 's|image: .*/notes-app-backend:.*|image: $BACKEND_IMAGE:${GIT_COMMIT}|' manifests-repo/ArgoCD-Pipeline/backend-deployment.yaml
                    """

                    sh '''
                    cd manifests-repo
                    git config user.name "jenkins"
                    git config user.email "jenkins@example.com"
                    git add ArgoCD-Pipeline/
                    git commit -m "CI: Update Notes App image tags to ${GIT_COMMIT}"
                    git push https://${GITHUB_TOKEN}@github.com/whosam1/argocd-pipeline.git main
                    '''
                }
            }
        }
    }
}
