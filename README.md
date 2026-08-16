pipeline {
	agent any
	environment {
		AWS_ACCESS_KEY_ID     = credentials('aws-access-key-id')
		AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
		AWS_DEFAULT_REGION    = 'us-east-1'
		DOCKER_REGISTRY       = ''
		DOCKER_CREDENTIALS_ID = ''
		GIT_CREDENTIALS_ID    = ''
		ARGOCD_SERVER         = ''
		ARGOCD_USERNAME       = ''
		ARGOCD_PASSWORD_ID    = ''
	}
	options {
		timestamps()
		// ansiColor('xterm')
		disableConcurrentBuilds()
	}
	stages {
		stage('Checkout') {
			steps {
				checkout scm
				script {
					COMMIT_SHORT = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
					IMAGE_TAG = "${env.BUILD_NUMBER}-${COMMIT_SHORT}"
				}
			}
		}

		stage('Setup AWS & Tools') {
			steps {
				sh '''
				set -e
				if ! command -v aws >/dev/null 2>&1; then
					pip install --user awscli || true
					export PATH=$HOME/.local/bin:$PATH
				fi
				if ! command -v kubectl >/dev/null 2>&1; then
					curl -sLO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
					chmod +x kubectl
					mv kubectl /usr/local/bin/ || mv kubectl $HOME/.local/bin/
				fi
				if ! command -v argocd >/dev/null 2>&1; then
					curl -sSL -o /tmp/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
					chmod +x /tmp/argocd
					mv /tmp/argocd /usr/local/bin/argocd || mv /tmp/argocd $HOME/.local/bin/argocd
				fi
				aws --version || true
				'''
				withEnv(["AWS_ACCESS_KEY_ID=${env.AWS_ACCESS_KEY_ID}", "AWS_SECRET_ACCESS_KEY=${env.AWS_SECRET_ACCESS_KEY}", "AWS_DEFAULT_REGION=${env.AWS_DEFAULT_REGION}"]) {
					sh 'echo AWS credentials configured'
				}
			}
		}

		stage('Authenticate to EKS') {
			steps {
				sh '''
				set -e
				if [ -n "${EKS_CLUSTER_NAME}" ]; then
					aws eks update-kubeconfig --name "${EKS_CLUSTER_NAME}" --region "${AWS_DEFAULT_REGION}"
				else
					echo 'EKS_CLUSTER_NAME not set; skipping kubeconfig update'
				fi
				kubectl version --client
				'''
			}
		}

		stage('Build and Push Images') {
			steps {
				script {
					def services = ['api-gateway','auth-service']
					withCredentials([usernamePassword(credentialsId: env.DOCKER_CREDENTIALS_ID, passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
						sh '''
						set -e
						if [ -n "${DOCKER_REGISTRY}" ]; then
							echo Logging in to registry
							echo ${DOCKER_PASS} | docker login ${DOCKER_REGISTRY} -u ${DOCKER_USER} --password-stdin
						fi
						'''
						for (svc in services) {
							def svcPath = "services/${svc}"
							if (fileExists(svcPath)) {
								def image = "${DOCKER_REGISTRY}/${svc}:${IMAGE_TAG}"
								sh "if [ -f ${svcPath}/Dockerfile ] || [ -f ${svcPath}/Dockerfile.tmpl ]; then docker build -t ${image} ${svcPath}; docker push ${image}; else echo 'No Dockerfile for ${svc}, skipping build'; fi"
							} else {
								echo "Service ${svc} not found, skipping"
							}
						}
					}
				}
			}
		}

		stage('Update Kustomize Overlays') {
			steps {
				script {
					def overlays = ['services/api-gateway/overlays/dev','services/api-gateway/overlays/prod','services/auth-service/overlays/dev','services/auth-service/overlays/prod']
					sh '''
					set -e
					for ov in ${overlays}; do true; done
					'''
					for (ov in overlays) {
						sh "if [ -d ${ov} ]; then git fetch origin; git checkout -b jenkins/image-update-${IMAGE_TAG} || true; fi"
						def svcName = ov.split('/')[1]
						def image = "${DOCKER_REGISTRY}/${svcName}:${IMAGE_TAG}"
						sh "if [ -d ${ov} ]; then kustomize edit set image ${svcName}=${image} -k ${ov} || (grep -R -l 'image: .*${svcName}' ${ov} | xargs -r sed -i 's|image: .*${svcName}.*|image: ${image}|g'); fi"
					}
				}
			}
		}

		stage('Commit and Push Manifest Changes') {
			steps {
				withCredentials([usernamePassword(credentialsId: env.GIT_CREDENTIALS_ID, passwordVariable: 'GIT_PASS', usernameVariable: 'GIT_USER')]) {
					sh '''
					set -e
					git config user.email "jenkins@example.com"
					git config user.name "jenkins"
					git add -A
					if git diff --staged --quiet; then
						echo 'No manifest changes to commit'
					else
						git commit -m "chore: update images to ${IMAGE_TAG} [ci skip]"
						git push origin HEAD
					fi
					'''
				}
			}
		}

		stage('Trigger ArgoCD Sync') {
			steps {
				script {
					withCredentials([usernamePassword(credentialsId: env.ARGOCD_PASSWORD_ID, passwordVariable: 'ARGOCD_PASS', usernameVariable: 'ARGOCD_USER')]) {
						sh '''
						set -e
						if [ -n "${ARGOCD_SERVER}" ]; then
							argocd login ${ARGOCD_SERVER} --username ${ARGOCD_USER} --password ${ARGOCD_PASS} --insecure || true
							argocd app list -o name | xargs -n1 argocd app sync || true
						else
							echo 'ARGOCD_SERVER not set; skipping ArgoCD sync'
						fi
						'''
					}
				}
			}
		}
	}
	post {
		always {
			sh 'docker logout ${DOCKER_REGISTRY} || true'
		}
	}
}
