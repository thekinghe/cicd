pipeline {
    agent none   // 不指定全局 agent，每个 stage 单独定义

    environment {
        HARBOR_URL      = 'harbor.local:31941'
        HARBOR_PROJECT  = 'image'
        K8S_NAMESPACE   = 'default'
        IMAGE_NAME      = "${HARBOR_URL}/${HARBOR_PROJECT}/myapp:${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            agent any   // 使用 Jenkins 控制器节点拉代码
            steps {
                checkout scm
            }
        }

        stage('Build & Push Image') {
            agent {
                kubernetes {
                    yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: kaniko-builder
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command:
    - /busybox/sleep
    args:
    - '99999'
    volumeMounts:
    - name: harbor-ca
      mountPath: /kaniko/ssl/certs/
      readOnly: true
    - name: docker-config
      mountPath: /kaniko/.docker/
      readOnly: true
  volumes:
  - name: harbor-ca
    configMap:
      name: harbor-ca-cert
  - name: docker-config
    secret:
      secretName: harbor-credentials
      items:
      - key: .dockerconfigjson
        path: config.json
"""
                }
            }
            steps {
                container('kaniko') {
                    sh """
                        /kaniko/executor \
                          --context=. \
                          --dockerfile=Dockerfile \
                          --destination=${IMAGE_NAME} \
                          --cache=true
                    """
                }
            }
        }

        stage('Deploy to K8s') {
            agent any   // 控制器节点必须有 kubectl 和 kubeconfig 凭证
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
            agent any
            steps {
                script {
                    // 请根据实际 Service 暴露情况修改地址
                    // 例如 NodePort: http://<任一节点IP>:30080/health
                    // 或者用 kubectl 动态获取
                    def nodeIP = sh(script: "kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}'", returnStdout: true).trim()
                    def nodePort = sh(script: "kubectl get svc myapp-svc -n ${K8S_NAMESPACE} -o jsonpath='{.spec.ports[0].nodePort}'", returnStdout: true).trim()
                    sh "curl -sf http://${nodeIP}:${nodePort}/health || exit 1"
                }
            }
        }
    }
}
