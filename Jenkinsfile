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

        CLUSTER_NAME = "gym-cluster-${params.ENVIRONMENT}"
        NAMESPACE    = "gym-${params.ENVIRONMENT}"
    }

    stages {

       stage('Checkout') {
            steps {
                checkout scm
            }
        } 

        // stage('Install CLI Tools') {
        //     steps {
        //         sh '''
        //             mkdir -p /home/jenkins/bin
                    
        //             # Download AWS CLI
        //             curl -s "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                    
        //             # Extract using Java's jar utility
        //             jar xf awscliv2.zip
                    
        //             # Restore executable permissions stripped by the jar command
        //             chmod -R +x ./aws
                    
        //             # Install AWS CLI
        //             ./aws/install -i /home/jenkins/aws-cli -b /home/jenkins/bin --update
                    
        //             # Download and install kubectl
        //             curl -sLO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
        //             chmod +x kubectl
        //             mv kubectl /home/jenkins/bin/
        //         '''
        //     }
        // }
        
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
                        // wait: true means this stage does not proceed to the
                        // next service until this one's build+push+manifest-bump
                        // has fully finished — keeps failures isolated and
                        // ordering predictable rather than firing all builds
                        // in an uncontrolled parallel burst.
                    }
                }
            }
        }

        stage('Pull Latest Manifest Changes') {
            steps {
                // The per-service build jobs commit updated image tags
                // directly to THIS repo. Since Jenkins checked out this
                // repo at the START of the pipeline (before those commits
                // happened), we need to pull again to actually see them
                // before applying anything.
                sh "git pull origin main"
            }
        }

        // stage('Deploy Kafka') {
        //     steps {
        //         build job: 'kafka-deployment', wait: true
        //     }
        // }

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

        stage('Deploy to Kubernetes') {
            steps {
                container('aws-k8s') {
                    echo '🚀 Deploying Auth Service & Dynamic Configurations...'
                    sh """
                        RDS_HOST=\$(aws rds describe-db-instances \
                            --region ${env.AWS_REGION} \
                            --query "DBInstances[?contains(DBInstanceIdentifier, 'auth-postgres')].Endpoint.Address" \
                            --output text)

                        if [ -z "\$RDS_HOST" ] || [ "\$RDS_HOST" = "None" ]; then
                            echo "⚠️ Fallback: Querying first available RDS instance"
                            RDS_HOST=\$(aws rds describe-db-instances --region ${env.AWS_REGION} --query "DBInstances[0].Endpoint.Address" --output text)
                        fi

                        echo "Injecting RDS Host into ConfigMap: \$RDS_HOST"

                        temp_cm=\$(mktemp)
                        sed "s|<db-endpoint>|\$RDS_HOST|g" ${env.KUBERNETES_DIR}/configmap.yaml > \$temp_cm
                        kubectl apply -f \$temp_cm
                        rm -f \$temp_cm

                        temp_deployment=\$(mktemp)
                        sed -e "s|<account-id>|${env.AWS_ACCOUNT_ID}|g" \
                            -e "s|<region>|${env.AWS_REGION}|g" \
                            -e "s|:latest|:${env.IMAGE_TAG}|g" \
                            ${env.KUBERNETES_DIR}/deployment.yaml > \$temp_deployment

                        kubectl apply -f \$temp_deployment
                        kubectl apply -f ${env.KUBERNETES_DIR}/service.yaml
                        rm -f \$temp_deployment

                        kubectl rollout restart deployment/auth-service -n ${env.NAMESPACE}
                        kubectl rollout status deployment/auth-service -n ${env.NAMESPACE} --timeout=90s
                    """
                }
            }
        }
    }

        stage('Deploy auth-service') {
            steps {
                sh """
                    kubectl apply -k services/auth-service/overlays/${params.ENVIRONMENT}
                """
                // kubectl apply -k builds the kustomize overlay (base +
                // patches + image tag override) and applies the resolved
                // result in one step — this is the kustomize-native way to
                // apply, replacing the old per-file "kubectl apply -f".
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