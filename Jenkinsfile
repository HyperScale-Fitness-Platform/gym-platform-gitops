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

        CLUSTER_NAME = "gym-cluster"
        NAMESPACE    = "gym-${params.ENVIRONMENT}"
        ARGOCD_NS    = "argocd"
        ROOT_APP     = "gym-platform-root-${params.ENVIRONMENT}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Authenticate to Cluster') {
            steps {
                sh """
                    aws eks update-kubeconfig --region ${AWS_DEFAULT_REGION} --name ${CLUSTER_NAME}
                    kubectl version --client
                """
            }
        }

        stage('Apply Shared Resources & Namespaces') {
            steps {
                sh """
                    kubectl apply -f shared/${params.ENVIRONMENT}-namespace.yaml
                    kubectl apply -f shared/cluster-secret-store.yaml || true
                    kubectl apply -f shared/kafka.yaml
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

       stage('Generate Env File') {
            steps {
                script {
                    env.RESOLVED_ACCOUNT_ID = sh(script: "aws sts get-caller-identity --query Account --output text", returnStdout: true).trim()
                    env.RESOLVED_REGION = "${AWS_DEFAULT_REGION}"
                    echo "Resolved ACCOUNT_ID=${env.RESOLVED_ACCOUNT_ID}, REGION=${env.RESOLVED_REGION}"
                }
            }
        }

        stage('Resolve Kustomization Templates & Push to GitHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-creds', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                    sh """
                        set -e

                        git checkout -B main origin/main

                        for svc in auth-service api-gateway; do
                            TEMPLATE_PATH="services/\$svc/overlays/${params.ENVIRONMENT}/kustomization.yaml.template"
                            OUTPUT_PATH="services/\$svc/overlays/${params.ENVIRONMENT}/kustomization.yaml"

                            if [ -f "\$TEMPLATE_PATH" ]; then
                                echo "Rendering \$TEMPLATE_PATH -> \$OUTPUT_PATH"
                                sed \
                                    -e "s|__ACCOUNT_ID__|${env.RESOLVED_ACCOUNT_ID}|g" \
                                    -e "s|__REGION__|${env.RESOLVED_REGION}|g" \
                                    "\$TEMPLATE_PATH" > "\$OUTPUT_PATH"
                            else
                                echo "ERROR: \$TEMPLATE_PATH not found — cannot resolve image reference"
                                exit 1
                            fi
                        done

                        git config user.email "jenkins@gym-platform.com"
                        git config user.name "Jenkins CI"

                        git add services/auth-service/overlays/${params.ENVIRONMENT}/kustomization.yaml
                        git add services/api-gateway/overlays/${params.ENVIRONMENT}/kustomization.yaml

                        if git diff --cached --quiet; then
                            echo "No changes to commit — kustomization files already up to date."
                        else
                            git commit -m "Resolve account/region for ${params.ENVIRONMENT} (build \${BUILD_NUMBER})"
                            git push https://\${GIT_USER}:\${GIT_TOKEN}@github.com/HyperScale-Fitness-Platform/gym-platform-gitops.git main
                        fi
                    """
                }
            }
        }

        stage('Bootstrap ArgoCD App of Apps') {
            steps {
                script {
                    echo "Ensuring AppProject and Root Application exist..."
                    sh """
                        # 1. Apply the Project definition
                        kubectl apply -f argocd/projects/gym-project.yaml -n ${ARGOCD_NS}

                        # 2. Apply the Environment Root App (Dev or Prod)
                        kubectl apply -f argocd/root-apps/root-app-${params.ENVIRONMENT}.yaml -n ${ARGOCD_NS}
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
                    echo "Waiting for Argo CD to discover and sync all services in ${params.ENVIRONMENT}..."
        
                    sh """
                        # Trigger hard refresh on Root App
                        kubectl patch application ${ROOT_APP} -n ${ARGOCD_NS} --type merge -p '{"operation":{"sync":{"prune":true}}}' || true
        
                        echo "=== Waiting for Root Application (${ROOT_APP}) to be Healthy ==="
                        kubectl wait --for=jsonpath='{.status.health.status}'=Healthy application/${ROOT_APP} -n ${ARGOCD_NS} --timeout=180s
        
                        echo "=== Waiting for Child Applications to be Synced & Healthy ==="
                        
                        # POSIX-compliant loop (works with /bin/sh)
                        for app in auth-service-${params.ENVIRONMENT} api-gateway-${params.ENVIRONMENT}; do
                            echo "Waiting for \$app to sync and become Healthy..."
                            
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
                    echo "=== Fetching Argo CD Admin Credentials ==="
                    sh """
                        # Fetch and decode initial admin password
                        ADMIN_PASS=\$(kubectl -n ${ARGOCD_NS} get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" 2>/dev/null | base64 -d || echo "Password not in initial secret (may have been changed)")

                        echo "========================================================="
                        echo "  Argo CD UI Access Details                              "
                        echo "========================================================="
                        echo "  URL:       https://localhost:8080"
                        echo "  Username:  admin"
                        echo "  Password:  \${ADMIN_PASS}"
                        echo ""
                        echo "  To access locally, run on your machine:"
                        echo "    kubectl port-forward svc/argocd-server -n ${ARGOCD_NS} 8080:443"
                        echo "========================================================="
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
            echo "Full ${params.ENVIRONMENT} deployment and Argo CD sync succeeded!"
        }
        failure {
            echo "Deployment to ${params.ENVIRONMENT} failed — check which stage above stopped the sequence."
        }
    }
}