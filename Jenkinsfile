pipeline {
    agent any
    environment {
        GITHUB_REPO = 'https://github.com/atchyuth417/python-hello-world-k8s.git'
        DOCKER_IMAGE = 'atchyuth417/hello-world'
        HELM_CHART_DIR = 'helm'
        KUBE_CONFIG = credentials('kubeconfig')
    }
    stages {
        stage('Checkout Code') {
            steps {
                git url: "${GITHUB_REPO}", branch: 'main'
            }
        }
        stage('Build Docker Image') {
            steps {
                script {
                    def image = docker.build("${DOCKER_IMAGE}:${env.BUILD_NUMBER}")
                }
            }
        }
        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-credentials') {
                        docker.image("${DOCKER_IMAGE}:${env.BUILD_NUMBER}").push()
                        docker.image("${DOCKER_IMAGE}:${env.BUILD_NUMBER}").push('latest')
                    }
                }
            }
        }
        stage('Install Helm (if needed)') {
            steps {
                script {
                    sh '''
                    if ! command -v helm &> /dev/null; then
                        echo "Helm not found, installing in /tmp..."
                        curl -fsSL -o helm.tar.gz https://get.helm.sh/helm-v3.17.2-linux-amd64.tar.gz
                        tar -zxvf helm.tar.gz
                        mv linux-amd64/helm /tmp/helm
                        chmod +x /tmp/helm
                        ls -l /tmp/helm  # Debug: Verify permissions
                        /tmp/helm version || echo "Failed to run /tmp/helm"
                    else
                        echo "Helm already installed"
                        helm version
                    fi
                    '''
                }
            }
        }
        stage('Update Helm Values') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github-credentials', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                    sh '''
                    ls -l helm/values.yaml  # Debug: Verify file exists
                    cat helm/values.yaml    # Debug: Check contents before
                    sed -i 's|tag: \\"latest\\"|tag: \\"${BUILD_NUMBER}\\"|' helm/values.yaml
                    cat helm/values.yaml    # Debug: Check contents after
                    git status              # Debug: Check Git status
                    git add helm/values.yaml
                    git status              # Debug: Confirm staging
                    git commit -m "Update image tag to ${BUILD_NUMBER}" || echo "No changes to commit"
                    git push origin main
                    '''
                }
            }
        }
        stage('Deploy to Kubernetes via Helm') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                    /tmp/helm upgrade --install hello-world helm --namespace default --kubeconfig "$KUBECONFIG"
                    '''
                }
            }
        }
    }
    post {
        success {
            echo 'Deployment successful!'
        }
        failure {
            echo 'Deployment failed!'
        }
    }
}
