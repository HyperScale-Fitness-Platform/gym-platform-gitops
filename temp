pipeline {
    agent any

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'prod'],
            description: 'Target environment overlay to update in GitOps'
        )
    }

    environment {
        ECR_REPO_NAME = "gym-api-gateway"
        NAMESPACE     = "gym-dev"
        AWS_REGION    = "us-east-1"

        // Safe evaluation fallback for Git SHA
        IMAGE_TAG     = "${env.GIT_COMMIT ? env.GIT_COMMIT.take(7) : 'latest'}"

        // AWS Credentials from Jenkins Store
        AWS_ACCESS_KEY_ID     = credentials('aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
        AWS_ACCOUNT_ID        = credentials('aws-account-id')

        GITOPS_REPO_URL = "https://github.com/HyperScale-Fitness-Platform/gym-platform-gitops.git"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            agent {
                docker { image 'node:20-alpine' }
            }
            steps {
                sh 'npm install'
            }
        }

        stage('ECR Authentication') {
            steps {
                echo '🔐 Authenticating Docker daemon with AWS ECR...'
                sh "aws ecr get-login-password --region ${env.AWS_REGION} | docker login --username AWS --password-stdin ${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com"
            }
        }

        stage('Build Container Image') {
            steps {
                echo "🏭 Building Docker image tagged as: ${env.IMAGE_TAG}..."
                sh """
                    docker build -t ${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com/${env.ECR_REPO_NAME}:${env.IMAGE_TAG} .
                    docker tag ${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com/${env.ECR_REPO_NAME}:${env.IMAGE_TAG} ${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com/${env.ECR_REPO_NAME}:latest
                """
            }
        }

        stage('Push Image to AWS ECR') {
            steps {
                echo "🚀 Pushing image artifact [${env.IMAGE_TAG}] to AWS ECR..."
                sh """
                    docker push ${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com/${env.ECR_REPO_NAME}:${env.IMAGE_TAG}
                    docker push ${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com/${env.ECR_REPO_NAME}:latest
                """
            }
        }

        stage('Update Image Tag in GitOps Repo') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-pat', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                    sh """
                        rm -rf gitops-repo
                        git clone https://${GIT_USER}:${GIT_TOKEN}@github.com/HyperScale-Fitness-Platform/gym-platform-gitops.git gitops-repo
                        
                        cd gitops-repo/services/api-gateway/overlays/${params.ENVIRONMENT}

                        # Update Kustomization image tag
                        kustomize edit set image ${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com/${env.ECR_REPO_NAME}=${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com/${env.ECR_REPO_NAME}:${env.IMAGE_TAG}

                        git config user.email "jenkins@gym-platform.com"
                        git config user.name "Jenkins CI"
                        
                        git add kustomization.yaml
                        git commit -m "ci(api-gateway): update ${params.ENVIRONMENT} image tag -> ${env.IMAGE_TAG}"
                        
                        # Rebase before pushing to prevent race condition conflicts if run in parallel
                        git pull --rebase origin main
                        git push origin main
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ api-gateway:${env.IMAGE_TAG} build complete and GitOps repo updated successfully!"
        }
        failure {
            echo "❌ Pipeline failed! Check the step diagnostics above."
        }
    }
}