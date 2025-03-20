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
                        /tmp/helm version
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
                    ls -l helm/values.yaml
                    cat helm/values.yaml
                    sed -i "s|tag: \\"[^\\"]*\\"|tag: \\"${BUILD_NUMBER}\\"|" helm/values.yaml || echo "sed failed"
                    cat helm/values.yaml
                    git status
                    git add helm/values.yaml
                    git commit -m "Update image tag to ${BUILD_NUMBER}" || echo "No changes to commit"
                    git config user.email "jenkins@example.com"
                    git config user.name "Jenkins"
                    git config credential.helper '!f() { echo username=$GIT_USERNAME; echo password=$GIT_PASSWORD; }; f'
                    git push origin main
                    '''
                }
            }
        }
        stage('Deploy to Kubernetes via Helm') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                    echo "Saving kubeconfig to file"
                    cat "$KUBECONFIG" > kubeconfig.yaml
                    ls -lh kubeconfig.yaml  # Check size
                    wc -l kubeconfig.yaml   # Check line count
                    echo "Rendering Helm chart"
                    /tmp/helm template hello-world helm --namespace default > rendered.yaml
                    ls -lh rendered.yaml
                    echo "Cleaning up any existing release"
                    /tmp/helm uninstall hello-world --namespace default --kubeconfig kubeconfig.yaml || echo "No existing release to uninstall"
                    echo "Deploying with Helm"
                    /tmp/helm upgrade --install hello-world helm --namespace default --kubeconfig kubeconfig.yaml --debug --atomic
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
