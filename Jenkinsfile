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
                        echo "Helm not found, installing in workspace..."
                        curl -fsSL -o helm.tar.gz https://get.helm.sh/helm-v3.17.2-linux-amd64.tar.gz
                        tar -zxvf helm.tar.gz
                        mv linux-amd64/helm ./helm
                        chmod +x ./helm
                        ls -l ./helm  # Debug: Check permissions
                        whoami        # Debug: Check running user
                        pwd           # Debug: Check workspace path
                        ./helm version || echo "Failed to run ./helm"
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
                    sh """
                    sed -i 's|tag: \\"latest\\"|tag: \\"${env.BUILD_NUMBER}\\"|' ${HELM_CHART_DIR}/values.yaml
                    git config user.email "jenkins@example.com"
                    git config user.name "Jenkins"
                    git config credential.helper '!f() { echo username=$GIT_USERNAME; echo password=$GIT_PASSWORD; }; f'
                    git add ${HELM_CHART_DIR}/values.yaml
                    git commit -m "Update image tag to ${env.BUILD_NUMBER}"
                    git push origin main
                    """
                }
            }
        }
        stage('Deploy to Kubernetes via Helm') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh "./helm upgrade --install hello-world ${HELM_CHART_DIR} --namespace default --kubeconfig $KUBECONFIG"
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
