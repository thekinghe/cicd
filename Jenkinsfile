pipeline {
    agent any

    environment {
        HARBOR_URL      = 'https://harbor.local:31941'  // 改为你的 Harbor 地址
        HARBOR_PROJECT  = 'image'                  // 你的 Harbor 项目名
        K8S_NAMESPACE   = 'default'                // 部署到的命名空间
        IMAGE_NAME      = "${HARBOR_URL}/${HARBOR_PROJECT}/myapp:${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build & Push Image') {
            steps {
                script {
                    docker.withRegistry("https://${HARBOR_URL}", 'harbor-creds') {
                        def app = docker.build(IMAGE_NAME)
                        app.push()
                    }
                }
            }
        }

        stage('Deploy to K8s') {
            steps {
                withKubeConfig([credentialsId: 'k8s-kubeconfig']) {
                    sh """
                        sed -i 's|IMAGE_PLACEHOLDER|${IMAGE_NAME}|' deploy/k8s-deployment.yaml
                        kubectl apply -n ${K8S_NAMESPACE} -f deploy/k8s-deployment.yaml
                        kubectl rollout status deployment/myapp -n ${K8S_NAMESPACE} --timeout=120s
                    """
                }
            }
        }

        stage('Smoke Test') {
            steps {
                // 注意：改成你的一个节点 IP 或 Service 的外部访问地址
                sh 'curl -sf http://10.0.0.100:30080/health || exit 1'
            }
        }
    }
}
