pipeline {
    agent {
        kubernetes {
            label 'jenkins-agent'
            defaultContainer 'build-tools'
        }
    }
    environment {
        NAMESPACE = "pipeline-${env.JOB_NAME.replaceAll('/', '-')}-${env.BUILD_NUMBER}"
        DOCKER_IMAGE = "avangelstech/staraus:${env.BUILD_NUMBER}"
        GIT_REPO = "https://github.com/avangels-tech/avangels-techin.git"
    }
    stages {
        stage('Checkout') {
            steps {
                git url: "${GIT_REPO}", branch: 'main'
            }
        }
        stage('Build and Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                    container('docker') {
                        sh 'docker build -t ${DOCKER_IMAGE} .'
                        sh 'echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin'
                        sh 'docker push ${DOCKER_IMAGE}'
                    }
                }
            }
        }
        stage('Create Namespace and RBAC') {
            steps {
                container('kubectl') {
                    sh """
                        kubectl create namespace ${NAMESPACE} || echo "Namespace ${NAMESPACE} already exists"
                    """
                    sh """
                        kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: ${NAMESPACE}
  name: jenkins-deployer
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: ${NAMESPACE}
  name: jenkins-deployer-binding
subjects:
- kind: ServiceAccount
  name: default
  namespace: jenkinsx
roleRef:
  kind: Role
  name: jenkins-deployer
  apiGroup: rbac.authorization.k8s.io
EOF
                    """
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                container('kubectl') {
                    sh """
                        sed -i "s|namespace: default|namespace: ${NAMESPACE}|g" k8s-deployment.yaml
                        sed -i "s|latest|${BUILD_NUMBER}|g" k8s-deployment.yaml
                        kubectl apply -f k8s-deployment.yaml
                    """
                }
            }
        }
    }
    post {
        always {
            echo "Pipeline completed!"
        }
    }
}
