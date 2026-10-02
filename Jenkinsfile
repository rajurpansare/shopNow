pipeline {
    agent any

    parameters {
        string(name: 'DOCKERHUB_USERNAME', defaultValue: 'YOUR_DOCKERHUB_USERNAME', description: 'Docker Hub username')
        string(name: 'K8S_NAMESPACE', defaultValue: 'shopnow', description: 'Kubernetes namespace')
        choice(name: 'DEPLOY', choices: ['true', 'false'], description: 'Deploy to Kubernetes after pushing images')
    }

    environment {
        IMAGE_TAG = "${BUILD_NUMBER}"
        DOCKER_CREDENTIALS = 'dockerhub-creds'
        KUBECONFIG_CREDENTIALS = 'shopnow-kubeconfig'
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Validate') {
            steps {
                sh '''
                  set -e
                  node --version
                  npm --version
                  test -f backend/package.json
                  test -f frontend/package.json
                  test -f admin/package.json
                '''
            }
        }

        stage('Install & Build') {
            parallel {
                stage('Backend') {
                    steps { sh 'cd backend && npm ci --omit=dev' }
                }
                stage('Frontend') {
                    steps { sh 'cd frontend && npm ci && npm run build' }
                }
                stage('Admin') {
                    steps { sh 'cd admin && npm ci && npm run build' }
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                  set -e
                  docker build -t ${DOCKERHUB_USERNAME}/shopnow-backend:${IMAGE_TAG} ./backend
                  docker build -t ${DOCKERHUB_USERNAME}/shopnow-frontend:${IMAGE_TAG} ./frontend
                  docker build -t ${DOCKERHUB_USERNAME}/shopnow-admin:${IMAGE_TAG} ./admin
                '''
            }
        }

        stage('Docker Push') {
            when { expression { params.DEPLOY == 'true' } }
            steps {
                withCredentials([usernamePassword(credentialsId: env.DOCKER_CREDENTIALS,
                    usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh '''
                      set -e
                      echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                      docker push ${DOCKERHUB_USERNAME}/shopnow-backend:${IMAGE_TAG}
                      docker push ${DOCKERHUB_USERNAME}/shopnow-frontend:${IMAGE_TAG}
                      docker push ${DOCKERHUB_USERNAME}/shopnow-admin:${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Deploy with Helm') {
            when { expression { params.DEPLOY == 'true' } }
            steps {
                withCredentials([file(credentialsId: env.KUBECONFIG_CREDENTIALS, variable: 'KCFG')]) {
                    sh '''
                      set -e
                      export KUBECONFIG="$KCFG"
                      kubectl cluster-info
                      helm upgrade --install shopnow kubernetes/helm/shopnow \\
                        --namespace ${K8S_NAMESPACE} --create-namespace \\
                        --set images.backend=${DOCKERHUB_USERNAME}/shopnow-backend:${IMAGE_TAG} \\
                        --set images.frontend=${DOCKERHUB_USERNAME}/shopnow-frontend:${IMAGE_TAG} \\
                        --set images.admin=${DOCKERHUB_USERNAME}/shopnow-admin:${IMAGE_TAG}
                      kubectl rollout status deployment/backend -n ${K8S_NAMESPACE} --timeout=180s
                      kubectl rollout status deployment/frontend -n ${K8S_NAMESPACE} --timeout=180s
                      kubectl rollout status deployment/admin -n ${K8S_NAMESPACE} --timeout=180s
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
            sh 'kubectl version --client || true'
            sh 'helm version || true'
        }
        success { echo 'ShopNow CI/CD pipeline completed successfully.' }
        failure { echo 'Pipeline failed. Check the stage log above for the first error.' }
    }
}
