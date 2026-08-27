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
        DUCKDNS_DOMAIN        = "iti-gym-platform"

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

                    if ! command -v aws >/dev/null 2>&1; then
                        echo "Installing AWS CLI v2..."
                        curl -s "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "/tmp/awscliv2.zip"
                        curl -sL "https://busybox.net/downloads/binaries/1.35.0-x86_64-linux-musl/busybox" -o /tmp/busybox
                        chmod +x /tmp/busybox
                        /tmp/busybox unzip -q -o /tmp/awscliv2.zip -d /tmp/
                        rm -f /tmp/busybox
                        /tmp/aws/install --install-dir "${WORKSPACE}/.tools/aws-cli" --bin-dir "${TOOL_BIN}" --update
                        rm -rf /tmp/aws /tmp/awscliv2.zip
                    fi

                    if ! command -v kubectl >/dev/null 2>&1; then
                        echo "Installing kubectl..."
                        curl -sLO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                        chmod +x kubectl
                        mv kubectl "${TOOL_BIN}/"
                    fi

                    if ! command -v envsubst >/dev/null 2>&1; then
                        echo "Installing envsubst..."
                        curl -sL https://github.com/a8m/envsubst/releases/download/v1.2.0/envsubst-`uname -s`-`uname -m` -o "${TOOL_BIN}/envsubst"
                        chmod +x "${TOOL_BIN}/envsubst"
                    fi

                    echo "--- Tool Versions & Checks ---"
                    aws --version
                    kubectl version --client
                    echo "envsubst available at: $(which envsubst)"
                '''
            }
        }
        
        stage('Authenticate to Cluster & Resolve Account') {
            steps {
                script {
                    sh "aws eks update-kubeconfig --region ${AWS_DEFAULT_REGION} --name ${CLUSTER_NAME}"

                    env.AWS_ACCOUNT_ID = sh(
                        script: "aws sts get-caller-identity --query Account --output text",
                        returnStdout: true
                    ).trim()
                    
                    env.ECR_REGISTRY = "${env.AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
                    echo "Connected to EKS. Resolved AWS Account: ${env.AWS_ACCOUNT_ID} | ECR: ${env.ECR_REGISTRY}"
                }
            }
        }

        stage('Apply Shared Resources & Namespaces') {
            steps {
                sh """
                    kubectl apply -f shared/${params.ENVIRONMENT}-namespace.yaml

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
                    def services = [
                        'kafka-service',
                        'gym-api-gateway',
                        'gym-auth-service',
                        'gym-profile-service',
                        'gym-social-service',
                        'gym-ai-service',
                        'frontend-service',
                        'gym-progress-service',
                        'gym-operations-service'
                    ]
                    services.each { jobName ->
                        echo "Triggering ${jobName} for environment ${params.ENVIRONMENT}..."
                        build job: jobName,
                              parameters: [string(name: 'ENVIRONMENT', value: params.ENVIRONMENT)],
                              wait: true
                    }
                }
            }
        }

        stage('Bootstrap ArgoCD Apps') {
            steps {
                sh """
                    kubectl apply -f argocd/projects/gym-project.yaml -n ${ARGOCD_NS}
                    kubectl apply -f argocd/root-apps/root-app-${params.ENVIRONMENT}.yaml -n ${ARGOCD_NS}
                """
            }
        }

        stage('Sync TLS Certificate to AWS ACM') {
            steps {
                sh """
                    echo "Waiting for TLS certificate secret 'api-gateway-tls' in ${NAMESPACE}..."
                    until kubectl get secret api-gateway-tls -n ${NAMESPACE} >/dev/null 2>&1; do
                        echo "Waiting for cert-manager to generate api-gateway-tls..."
                        sleep 5
                    done

                    # 1. Extract secret
                    kubectl get secret api-gateway-tls -n ${NAMESPACE} -o jsonpath="{.data['tls\\\\.crt']}" | base64 -d > /tmp/fullchain.crt
                    kubectl get secret api-gateway-tls -n ${NAMESPACE} -o jsonpath="{.data['tls\\\\.key']}" | base64 -d > /tmp/tls.key

                    # 2. Extract leaf certificate
                    awk '/-----BEGIN CERTIFICATE-----/{flag=1; count++} count==1{print} /-----END CERTIFICATE-----/{flag=0; if(count==1) exit}' /tmp/fullchain.crt > /tmp/leaf.crt

                    # 3. Extract CA chain
                    awk '/-----BEGIN CERTIFICATE-----/{count++} count>1{print}' /tmp/fullchain.crt > /tmp/chain.crt

                    # 4. Check for existing ACM certificate or import new
                    EXISTING_ARN=\$(aws acm list-certificates --region ${AWS_DEFAULT_REGION} \
                      --query "CertificateSummaryList[?DomainName=='${DUCKDNS_DOMAIN}.duckdns.org'].CertificateArn | [0]" \
                      --output text)

                    if [ "\${EXISTING_ARN}" != "None" ] && [ -n "\${EXISTING_ARN}" ]; then
                        echo "Updating existing ACM Certificate: \${EXISTING_ARN}"
                        CERT_ARN=\$(aws acm import-certificate \
                          --certificate-arn "\${EXISTING_ARN}" \
                          --certificate fileb:///tmp/leaf.crt \
                          --certificate-chain fileb:///tmp/chain.crt \
                          --private-key fileb:///tmp/tls.key \
                          --region ${AWS_DEFAULT_REGION} \
                          --query "CertificateArn" \
                          --output text)
                    else
                        echo "Importing new ACM Certificate..."
                        CERT_ARN=\$(aws acm import-certificate \
                          --certificate fileb:///tmp/leaf.crt \
                          --certificate-chain fileb:///tmp/chain.crt \
                          --private-key fileb:///tmp/tls.key \
                          --region ${AWS_DEFAULT_REGION} \
                          --query "CertificateArn" \
                          --output text)
                    fi

                    echo "Successfully configured ACM ARN: \${CERT_ARN}"

                    # 5. Annotate Ingress with the valid ACM ARN
                    kubectl annotate ingress api-gateway-ingress -n ${NAMESPACE} \
                      alb.ingress.kubernetes.io/certificate-arn="\${CERT_ARN}" --overwrite

                    # 6. Cleanup temporary files
                    rm -f /tmp/fullchain.crt /tmp/leaf.crt /tmp/chain.crt /tmp/tls.key
                """
            }
        }

        stage('Sync & Watch ArgoCD Deployment') {
            when {
                expression { params.SYNC_ARGOCD == true }
            }
            steps {
                sh """
                    # Trigger refresh on Root App
                    kubectl patch application ${ROOT_APP} -n ${ARGOCD_NS} --type merge -p '{"operation":{"sync":{"prune":true}}}' || true

                    echo "=== Waiting for Root Application (${ROOT_APP}) to become Healthy ==="
                    kubectl wait --for=jsonpath='{.status.health.status}'=Healthy application/${ROOT_APP} -n ${ARGOCD_NS} --timeout=180s

                    echo "=== Waiting for Child Applications to Sync & Become Healthy ==="
                    for app in auth-service-${params.ENVIRONMENT} api-gateway-${params.ENVIRONMENT} \
                               profile-service-${params.ENVIRONMENT} social-service-${params.ENVIRONMENT} \
                               ai-service-${params.ENVIRONMENT} frontend-service-${params.ENVIRONMENT} \
                               progress-service-${params.ENVIRONMENT} kafka-service-${params.ENVIRONMENT}; do
                        echo "Checking status for \$app..."
                        kubectl wait --for=jsonpath='{.status.sync.status}'=Synced application/\$app -n ${ARGOCD_NS} --timeout=180s || true
                        kubectl wait --for=jsonpath='{.status.health.status}'=Healthy application/\$app -n ${ARGOCD_NS} --timeout=300s
                    done
                """
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

        stage('Full System Smoke Test & DNS Sync') {
            steps {
                script {
                    echo "Waiting for Ingress ALB hostname..."
                    def albHostname = ""
                    timeout(time: 5, unit: 'MINUTES') {
                        waitUntil {
                            albHostname = sh(
                                script: "kubectl get ingress api-gateway-ingress -n ${NAMESPACE} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || true",
                                returnStdout: true
                            ).trim()
                            return (albHostname != null && albHostname != "")
                        }
                    }
        
                    echo "ALB Hostname: ${albHostname}"
        
                    withCredentials([string(credentialsId: 'duckdns-token', variable: 'DUCKDNS_TOKEN')]) {
                        sh """
                            echo "Resolving ALB IP address..."
                            ALB_IP=\$(getent hosts ${albHostname} | awk '{ print \$1 }' | head -n 1)
        
                            if [ -z "\${ALB_IP}" ]; then
                                echo "Primary lookup empty, trying Google DNS..."
                                ALB_IP=\$(nslookup ${albHostname} 8.8.8.8 | grep -A1 'Name:' | grep 'Address:' | awk '{print \$2}' | head -n 1)
                            fi
        
                            echo "Resolved ALB IP: \${ALB_IP}"
        
                            if [ -n "\${ALB_IP}" ]; then
                                echo "Updating DuckDNS record for ${DUCKDNS_DOMAIN}..."
                                UPDATE_RESP=\$(curl -s "https://www.duckdns.org/update?domains=${DUCKDNS_DOMAIN}&token=\${DUCKDNS_TOKEN}&ip=\${ALB_IP}")
                                echo "DuckDNS Response: \${UPDATE_RESP}"
                            else
                                echo "WARNING: Could not resolve ALB IP. DuckDNS update skipped."
                            fi
                        """
                    }
        
                    echo "Testing HTTPS Health endpoint..."
                    sh "curl -sf https://${DUCKDNS_DOMAIN}.duckdns.org/health"
        
                    echo "Testing Auth Registration endpoint..."
                    sh """
                        curl -sf -X POST https://${DUCKDNS_DOMAIN}.duckdns.org/auth/register \
                          -H "Content-Type: application/json" \
                          -d '{"email":"smoketest-${env.BUILD_NUMBER}@test.com","password":"secret123","role":"customer"}'
                    """
                }
            }
        }
    }

    post {
        always {
            deleteDir()
        }
        success {
            echo "✅ Full ${params.ENVIRONMENT} deployment, ACM sync, and DuckDNS update active!"
        }
        failure {
            echo "❌ Deployment to ${params.ENVIRONMENT} failed. Check stage logs for details."
        }
    }
}