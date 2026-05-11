pipeline {
    agent none

    environment {
        HARBOR_URL      = 'harbor.local:31941'
        HARBOR_PROJECT  = 'image'
        K8S_NAMESPACE   = 'default'
        IMAGE_NAME      = "${HARBOR_URL}/${HARBOR_PROJECT}/myapp:${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            agent any
            steps {
                checkout scm
                stash includes: '**/*', name: 'source', useDefaultExcludes: false
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
  securityContext:
    runAsUser: 0
  hostAliases:
  - ip: "192.168.1.220"          # ← 改成你的 Harbor 所在节点 IP
    hostnames:
    - "harbor.local"
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
  - name: jnlp
    image: jenkins/inbound-agent:3355.v388858a_47b_33-3-jdk21
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
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
                unstash 'source'
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
            agent {
                kubernetes {
                    yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: kubectl-deployer
spec:
  containers:
  - name: kubectl
    image: alpine/k8s:1.27.8   # 同时带 kubectl 和 curl
    command:
    - sleep
    args:
    - '99999'
  - name: jnlp
    image: jenkins/inbound-agent:3355.v388858a_47b_33-3-jdk21
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
"""
                }
            }
            steps {
                unstash 'source'
                container('kubectl') {
                    withCredentials([file(credentialsId: 'k8s-cred', variable: 'KUBECONFIG_FILE')]) {
                        sh """
                            sed -i 's|IMAGE_PLACEHOLDER|${IMAGE_NAME}|' deploy/k8s-deployment.yaml
                            kubectl --kubeconfig="${KUBECONFIG_FILE}" apply -n ${K8S_NAMESPACE} -f deploy/k8s-deployment.yaml
                            kubectl --kubeconfig="${KUBECONFIG_FILE}" rollout status deployment/myapp -n ${K8S_NAMESPACE} --timeout=120s
                        """
                    }
                }
            }
        }

        stage('Smoke Test') {
            agent {
                kubernetes {
                    yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: kubectl-smoke
spec:
  containers:
  - name: kubectl
    image: alpine/k8s:1.27.8
    command:
    - sleep
    args:
    - '99999'
  - name: jnlp
    image: jenkins/inbound-agent:3355.v388858a_47b_33-3-jdk21
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
"""
                }
            }
            steps {
                container('kubectl') {
                    withCredentials([file(credentialsId: 'k8s-cred', variable: 'KUBECONFIG_FILE')]) {
                        script {
                            def nodeIP = sh(script: "kubectl --kubeconfig=\"${KUBECONFIG_FILE}\" get nodes -o jsonpath='{.items[0].status.addresses[0].address}'", returnStdout: true).trim()
                            def nodePort = sh(script: "kubectl --kubeconfig=\"${KUBECONFIG_FILE}\" get svc myapp-svc -n ${K8S_NAMESPACE} -o jsonpath='{.spec.ports[0].nodePort}'", returnStdout: true).trim()
                            sh "curl -sf http://${nodeIP}:${nodePort}/health || exit 1"
                        }
                    }
                }
            }
        }
    }
}
