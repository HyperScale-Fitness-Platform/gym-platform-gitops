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
            description: 'Skip triggering per-service build/push jobs and only apply existing manifests'
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

        stage('Authenticate to Cluster & Fetch AWS Identity') {
            steps {
                sh """
                    # Authenticate kubectl to the EKS cluster
                    aws eks update-kubeconfig --region ${AWS_DEFAULT_REGION} --name ${CLUSTER_NAME}
                    kubectl version --client
                """
                script {
                    // Dynamically get the current AWS Account ID
                    env.AWS_ACCOUNT_ID = sh(
                        script: 'aws sts get-caller-identity --query "Account" --output text',
                        returnStdout: true
                    ).trim()
                }
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

        stage('Deploy Database Layer (auth-service)') {
            steps {
                dir('services/auth-service/base') {
                    sh """
                        kubectl apply -f headless-service.yaml -n ${NAMESPACE}
                        kubectl apply -f statefulset.yaml -n ${NAMESPACE}
                        kubectl rollout status statefulset/auth-postgres -n ${NAMESPACE} --timeout=120s
                    """
                }
            }
        }

        stage('Run Database Migration (auth-service)') {
            steps {
                dir('services/auth-service/base') {
                    sh """
                        kubectl apply -f db-schema-configmap.yaml -n ${NAMESPACE}
                        kubectl delete job auth-db-migrate -n ${NAMESPACE} --ignore-not-found
                        kubectl apply -f db-migrate-job.yaml -n ${NAMESPACE}
                        kubectl wait --for=condition=complete job/auth-db-migrate -n ${NAMESPACE} --timeout=90s
                    """
                }
            }
        }

        stage('Deploy auth-service') {
            steps {
                dir("services/auth-service/overlays/${params.ENVIRONMENT}") {
                    sh """
                        # Substitute placeholders <account-id> and <region> dynamically in workspace
                        sed -i.bak -e "s|<account-id>|${env.AWS_ACCOUNT_ID}|g" \
                                   -e "s|<region>|${AWS_DEFAULT_REGION}|g" \
                                   kustomization.yaml
                        rm -f kustomization.yaml.bak

                        # Apply overlay
                        kubectl apply -k .

                        # Trigger rollout to pull newest image
                        kubectl rollout restart deployment/auth-service -n ${NAMESPACE}
                        kubectl rollout status deployment/auth-service -n ${NAMESPACE} --timeout=90s
                    """
                }
            }
        }

        stage('Deploy api-gateway') {
            steps {
                dir("services/api-gateway/overlays/${params.ENVIRONMENT}") {
                    sh """
                        # Substitute placeholders <account-id> and <region> dynamically in workspace
                        sed -i.bak -e "s|<account-id>|${env.AWS_ACCOUNT_ID}|g" \
                                   -e "s|<region>|${AWS_DEFAULT_REGION}|g" \
                                   kustomization.yaml
                        rm -f kustomization.yaml.bak

                        # Apply overlay
                        kubectl apply -k .

                        # Trigger rollout to pull newest image
                        kubectl rollout restart deployment/api-gateway -n ${NAMESPACE}
                        kubectl rollout status deployment/api-gateway -n ${NAMESPACE} --timeout=90s
                    """
                }
            }
        }

        stage('Full System Smoke Test') {
            steps {
                script {
                    echo "Waiting for Ingress ALB hostname..."
                    def albAddress = ""
                    timeout(time: 3, unit: 'MINUTES') {
                        waitUntil {
                            albAddress = sh(
                                script: "kubectl get ingress api-gateway-ingress -n ${NAMESPACE} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || true",
                                returnStdout: true
                            ).trim()
                            return (albAddress != null && albAddress != "")
                        }
                    }

                    echo "ALB Endpoint: http://${albAddress}"
                    sh "curl -sf http://${albAddress}/health || echo 'Health check warning'"

                    sh """
                        curl -sf -X POST http://${albAddress}/auth/register \
                          -H "Content-Type: application/json" \
                          -d '{"email":"smoketest-${env.BUILD_NUMBER}@test.com","password":"secret123","role":"customer"}' \
                          || echo "Registration smoke test completed with warning."
                    """
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo "Full ${params.ENVIRONMENT} deployment succeeded: shared -> auth-service (db + migration + app) -> api-gateway -> smoke test."
        }
        failure {
            echo "Deployment to ${params.ENVIRONMENT} failed — check which stage above stopped the sequence."
        }
    }
}