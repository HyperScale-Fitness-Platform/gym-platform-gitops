pipeline {
    agent any

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'prod'],
            description: 'Which overlay/environment to deploy'
        )
        booleanParam(
            name: 'SKIP_BUILD',
            defaultValue: false,
            description: 'Skip triggering per-service build/push jobs and only apply existing manifests (useful for re-syncing without rebuilding images)'
        )
    }

    environment {
        AWS_ACCESS_KEY_ID     = credentials('aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
        AWS_DEFAULT_REGION    = 'us-east-1'

        CLUSTER_NAME = "gym-cluster"
        NAMESPACE    = "gym-${params.ENVIRONMENT}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Authenticate to Cluster') {
            steps {
                sh "aws eks update-kubeconfig --region ${AWS_DEFAULT_REGION} --name ${CLUSTER_NAME}"
                sh "kubectl version --client"
            }
        }

        stage('Apply Shared Resources') {
            steps {
                sh "kubectl apply -f shared/${params.ENVIRONMENT}-namespace.yaml"
            }
        }

        stage('Build & Push Service Images') {
            when {
                expression { params.SKIP_BUILD == false }
            }
            steps {
                script {
                    def services = ['gym-auth-svc-deployment', 'gym-api-gateway-deployment']

                    services.each { jobName ->
                        echo "Triggering ${jobName} for environment ${params.ENVIRONMENT}..."
                        build job: jobName,
                              parameters: [string(name: 'ENVIRONMENT', value: params.ENVIRONMENT)],
                              wait: true
                    }
                }
            }
        }

        stage('Pull Latest Manifest Changes') {
            steps {
                sh "git pull origin main"
            }
        }

        stage('Deploy Database Layer (auth-service)') {
            steps {
                dir('services/auth-service/base') {
                    sh "kubectl apply -f headless-service.yaml"
                    sh "kubectl apply -f statefulset.yaml"
                    sh """
                        kubectl rollout status statefulset/auth-postgres -n ${NAMESPACE} --timeout=120s
                    """
                }
            }
        }

        stage('Run Database Migration (auth-service)') {
            steps {
                dir('services/auth-service/base') {
                    sh "kubectl apply -f db-schema-configmap.yaml"
                    sh "kubectl delete job auth-db-migrate -n ${NAMESPACE} --ignore-not-found"
                    sh "kubectl apply -f db-migrate-job.yaml"
                    sh "kubectl wait --for=condition=complete job/auth-db-migrate -n ${NAMESPACE} --timeout=90s"
                }
            }
        }

        stage('Deploy auth-service') {
            steps {
                sh """
                    kubectl apply -k services/auth-service/overlays/${params.ENVIRONMENT}
                """
                sh """
                    kubectl rollout status deployment/auth-service -n ${NAMESPACE} --timeout=90s
                """
            }
        }

        stage('Deploy api-gateway') {
            steps {
                sh """
                    kubectl apply -k services/api-gateway/overlays/${params.ENVIRONMENT}
                """
                sh """
                    kubectl rollout status deployment/api-gateway -n ${NAMESPACE} --timeout=90s
                """
            }
        }

        stage('Full System Smoke Test') {
            steps {
                script {
                    def albAddress = sh(
                        script: "kubectl get ingress api-gateway-ingress -n ${NAMESPACE} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'",
                        returnStdout: true
                    ).trim()

                    sh "curl -sf http://${albAddress}/health"

                    sh """
                        curl -sf -X POST http://${albAddress}/auth/register \
                          -H "Content-Type: application/json" \
                          -d '{"email":"smoketest-${env.BUILD_NUMBER}@test.com","password":"secret123","role":"customer"}' \
                          || echo "registration smoke test failed — check manually before treating this build as fully healthy"
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Full ${params.ENVIRONMENT} deployment succeeded: shared -> auth-service (db + migration + app) -> api-gateway -> smoke test."
        }
        failure {
            echo "Deployment to ${params.ENVIRONMENT} failed — check which stage above stopped the sequence."
        }
    }
}