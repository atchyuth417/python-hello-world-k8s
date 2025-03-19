pipeline {
    agent any
    environment {
        GITHUB_REPO = 'https://github.com/atchyuth417/python-hello-world-k8s.git'
        DOCKER_IMAGE = 'atchyuth417/hello-world'
        HELM_CHART_DIR = 'helm'
        KUBE_CONFIG = credentials('kubeconfig') // Jenkins credential for kubeconfig
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
                sh 'curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3'
                sh 'chmod +x get_helm.sh && ./get_helm.sh'
            }
        }
        stage('Update Helm Values') {
            steps {
                script {
                    sh """
                    sed -i 's|tag: \\"latest\\"|tag: \\"${env.BUILD_NUMBER}\\"|' ${HELM_CHART_DIR}/values.yaml
                    git config user.email "jenkins@example.com"
                    git config user.name "Jenkins"
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
                    sh "helm upgrade --install hello-world ${HELM_CHART_DIR} --namespace default --kubeconfig $KUBECONFIG"
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
