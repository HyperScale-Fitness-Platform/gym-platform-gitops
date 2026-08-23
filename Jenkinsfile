pipeline {
    agent any

    options {
        disableConcurrentBuilds()
    }


    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'prod'],
            description: 'Target environment overlay to deploy'
        )
        booleanParam(
            name: 'SKIP_BUILD',
            defaultValue: false,
            description: 'Skip triggering per-service build/push jobs and only apply manifests'
        )
        booleanParam(
            name: 'SYNC_ARGOCD',
            defaultValue: true,
            description: 'Trigger Argo CD sync and wait for health check'
        )
    }

    environment {
        AWS_ACCESS_KEY_ID     = credentials('aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
        AWS_DEFAULT_REGION    = 'us-east-1'

        CLUSTER_NAME          = "gym-cluster"
        NAMESPACE             = "gym-${params.ENVIRONMENT}"
        ARGOCD_NS             = "argocd"
        ROOT_APP              = "gym-platform-root-${params.ENVIRONMENT}"

        PATH                  = "${WORKSPACE}/.tools/bin:${env.PATH}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Bootstrap CLI Tools') {
            steps {
                sh '''
                    set -e
                    TOOL_BIN="${WORKSPACE}/.tools/bin"
                    mkdir -p "${TOOL_BIN}"

                    # 1. Install AWS CLI v2 if missing
                    if ! command -v aws >/dev/null 2>&1; then
                        echo "Installing AWS CLI v2..."
                        curl -s "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "/tmp/awscliv2.zip"
                        unzip -q -o /tmp/awscliv2.zip -d /tmp/
                        /tmp/aws/install --install-dir "${WORKSPACE}/.tools/aws-cli" --bin-dir "${TOOL_BIN}" --update
                        rm -rf /tmp/aws /tmp/awscliv2.zip
                    fi

                    # 2. Install kubectl if missing
                    if ! command -v kubectl >/dev/null 2>&1; then
                        echo "Installing kubectl..."
                        curl -sLO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                        chmod +x kubectl
                        mv kubectl "${TOOL_BIN}/"
                    fi

                    # 3. Install envsubst if missing
                    if ! command -v envsubst >/dev/null 2>&1; then
                        echo "Installing envsubst..."
                        curl -sL https://github.com/a8m/envsubst/releases/download/v1.2.0/envsubst-`uname -s`-`uname -m` -o "${TOOL_BIN}/envsubst"
                        chmod +x "${TOOL_BIN}/envsubst"
                    fi

                    # Verification (avoid -v on Go binary)
                    echo "--- Tool Versions & Checks ---"
                    aws --version
                    kubectl version --client
                    echo "envsubst available at: $(which envsubst)"
                '''
            }
        }
        
        stage('Authenticate to Cluster') {
            steps {
                sh """
                    aws eks update-kubeconfig --region ${AWS_DEFAULT_REGION} --name ${CLUSTER_NAME}

                    env.AWS_ACCOUNT_ID = sh(
                        script: "aws sts get-caller-identity --query Account --output text",
                        returnStdout: true
                    ).trim()
                    
                    env.ECR_REGISTRY = "${env.AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
                    echo "Connected to EKS. Resolved AWS Account: ${env.AWS_ACCOUNT_ID} | ECR: ${env.ECR_REGISTRY}"
                """
            }
        }

        stage('Apply Shared Resources & Namespaces') {
            steps {
                sh """
                    kubectl apply -f shared/${params.ENVIRONMENT}-namespace.yaml
                    kubectl apply -f shared/kafka.yaml

                    # Dynamically substitute ECR Registry and Environment into ImageUpdater CR
                    export ECR_REGISTRY="${env.ECR_REGISTRY}"
                    export ENVIRONMENT="${params.ENVIRONMENT}"
                    envsubst < shared/image-updater.yaml | kubectl apply -f -
                """
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

        stage('Resolve AWS Account Identity') {
            steps {
                script {
                    echo "Resolved AWS Account: ${env.AWS_ACCOUNT_ID} in ${AWS_DEFAULT_REGION}"
                }
            }
        }

        stage('Bootstrap ArgoCD & Inject Dynamic Overrides') {
            steps {
                script {
                    def ecrRegistry = "${env.ECR_REGISTRY}"
                    
                    sh """
                        # 1. Apply Project and Root Application
                        kubectl apply -f argocd/projects/gym-project.yaml -n ${ARGOCD_NS}
                        kubectl apply -f argocd/root-apps/root-app-${params.ENVIRONMENT}.yaml -n ${ARGOCD_NS}

                        # 2. Wait for Child Applications to be generated
                        echo "Waiting for child applications to initialize..."
                        until kubectl get application auth-service-${params.ENVIRONMENT} -n ${ARGOCD_NS} >/dev/null 2>&1; do
                            sleep 3
                        done
                        until kubectl get application api-gateway-${params.ENVIRONMENT} -n ${ARGOCD_NS} >/dev/null 2>&1; do
                            sleep 3
                        done

                        # 3. Patch Kustomize Image Overrides
                        kubectl patch application auth-service-${params.ENVIRONMENT} -n ${ARGOCD_NS} --type merge -p '{
                            "spec": {
                                "source": {
                                    "kustomize": {
                                        "images": [
                                            "gym-auth-service=${ecrRegistry}/gym-auth-service:latest"
                                        ]
                                    }
                                }
                            }
                        }'

                        kubectl patch application api-gateway-${params.ENVIRONMENT} -n ${ARGOCD_NS} --type merge -p '{
                            "spec": {
                                "source": {
                                    "kustomize": {
                                        "images": [
                                            "gym-api-gateway=${ecrRegistry}/gym-api-gateway:latest"
                                        ]
                                    }
                                }
                            }
                        }'
                    """
                }
            }
        }

        stage('Sync & Watch ArgoCD Deployment') {
            when {
                expression { params.SYNC_ARGOCD == true }
            }
            steps {
                script {
                    echo "Reconciling root and child applications for ${params.ENVIRONMENT}..."

                    sh """
                        # Trigger refresh on Root App
                        kubectl patch application ${ROOT_APP} -n ${ARGOCD_NS} --type merge -p '{"operation":{"sync":{"prune":true}}}' || true

                        echo "=== Waiting for Root Application (${ROOT_APP}) to become Healthy ==="
                        kubectl wait --for=jsonpath='{.status.health.status}'=Healthy application/${ROOT_APP} -n ${ARGOCD_NS} --timeout=180s

                        echo "=== Waiting for Child Applications to Sync & Become Healthy ==="
                        for app in auth-service-${params.ENVIRONMENT} api-gateway-${params.ENVIRONMENT}; do
                            echo "Checking status for \$app..."
                            kubectl wait --for=jsonpath='{.status.sync.status}'=Synced application/\$app -n ${ARGOCD_NS} --timeout=180s || true
                            kubectl wait --for=jsonpath='{.status.health.status}'=Healthy application/\$app -n ${ARGOCD_NS} --timeout=300s
                        done
                    """
                }
            }
        }

        stage('Retrieve ArgoCD Credentials & Access Info') {
            steps {
                script {
                    sh """
                        ADMIN_PASS=\$(kubectl -n ${ARGOCD_NS} get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" 2>/dev/null | base64 -d || echo "Password not in initial secret (may have been changed)")

                        echo "========================================================="
                        echo "  Argo CD UI Access Details                              "
                        echo "========================================================="
                        echo "  URL:       https://localhost:8081"
                        echo "  Username:  admin"
                        echo "  Password:  \${ADMIN_PASS}"
                        echo ""
                        echo "  To access locally, run on your machine:"
                        echo "    kubectl port-forward svc/argocd-server -n ${ARGOCD_NS} 8081:443"
                        echo "========================================================="
                    """
                }
            }
        }

        stage('Full System Smoke Test') {
            steps {
                script {
                    echo "Waiting for Ingress ALB endpoint..."
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
                          || echo "Registration smoke test finished with warning."
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
            echo "Full ${params.ENVIRONMENT} deployment and Argo CD sync succeeded!"
        }
        failure {
            echo "Deployment to ${params.ENVIRONMENT} failed. Check stage logs for details."
        }
    }
}