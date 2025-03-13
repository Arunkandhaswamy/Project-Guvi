pipeline {
    agent any

    environment {
        DOCKER_DEV_IMAGE = "arunkandhaswamy/dev-react-app"
        DOCKER_PROD_IMAGE = "arunkandhaswamy/prod-react-app"
        DOCKER_CREDENTIALS = "docker-hub-credentials"
        EC2_USER = "ubuntu"
        EC2_IP = "18.246.214.145"
    }

    triggers {
        githubPush()
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'prod', url: 'https://github.com/Arunkandhaswamy/Project-Guvi.git'
                script {
                    env.BRANCH_NAME = sh(script: "git rev-parse --abbrev-ref HEAD", returnStdout: true).trim()
                    echo "Branch Name Set: ${env.BRANCH_NAME}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "Executing build script for ${env.BRANCH_NAME}..."
                    sh './build.sh'
                    
                    if (env.BRANCH_NAME == 'dev') {
                        sh "docker tag devops-build_react-app:latest $DOCKER_DEV_IMAGE:latest"
                    } else if (env.BRANCH_NAME == 'prod') {
                        sh "docker tag devops-build_react-app:latest $DOCKER_PROD_IMAGE:latest"
                    }
                }
            }
        }

        stage('Login to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
                }
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                script {
                    echo "Branch Name: ${env.BRANCH_NAME}"
                    sh "docker images"
                    if (env.BRANCH_NAME == 'dev') {
                        echo "Pushing to public dev repository..."
                        sh "docker push $DOCKER_DEV_IMAGE:latest"
                    } else if (env.BRANCH_NAME == 'prod') {
                        echo "Checking if dev was merged before pushing to prod..."
                        def mergeCheck = sh(script: "git log --oneline -n 1 | grep 'Merge pull request'", returnStatus: true)
                        echo "Merge Check Exit Code: ${mergeCheck}"
                        if (mergeCheck == 0) {
                            echo "Dev branch was merged. Pushing to private prod repository..."
                            sh "docker push $DOCKER_PROD_IMAGE:latest"
                        } else {
                            echo "No merge detected. Skipping push to prod."
                        }
                    }
                }
            }
        }

        stage('Deploy to EC2') {
            when {
                anyOf {
                    branch 'dev'
                    branch 'prod'
                }
            }
            steps {
                     script {
                        def imageName = (env.BRANCH_NAME == 'prod') ? DOCKER_DEV_IMAGE : DOCKER_PROD_IMAGE
                        echo "Deploying $imageName to EC2..."
                        sh """
                            docker stop devops-build_react-app_1 || true &&
                            docker rm devops-build_react-app_1|| true &&
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin &&
                            docker pull $imageName:latest &&
                           
                             chmod +x deploy.sh
                            ./deploy.sh
                            
                        """
                    
                }
            }
        }
    }

    post {
        success { echo 'Deployment Successful!' }
        failure { echo 'Deployment Failed. Check Logs!' }
    }
}
