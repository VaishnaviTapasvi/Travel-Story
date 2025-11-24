pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:

  - name: node
    image: node:18
    command: ['cat']
    tty: true

  - name: sonar-scanner
    image: sonarsource/sonar-scanner-cli
    command: ['cat']
    tty: true

  - name: kubectl
    image: bitnami/kubectl:latest
    command: ['cat']
    tty: true
    env:
    - name: KUBECONFIG
      value: /kube/config
    volumeMounts:
    - name: kubeconfig-secret
      mountPath: /kube/config
      subPath: kubeconfig

  - name: dind
    image: docker:dind
    args: ["--storage-driver=overlay2"]
    securityContext:
      privileged: true
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""

  volumes:
  - name: kubeconfig-secret
    secret:
      secretName: kubeconfig-secret
'''
        }
    }

    environment {
        // Nexus registry hostname (used repeatedly) — edit if your env differs
        NEXUS_REGISTRY = "nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085"
        APP_NAMESPACE  = "2401198"
        FRONTEND_IMAGE = "${NEXUS_REGISTRY}/2401198/travelstory-frontend:v1"
        BACKEND_IMAGE  = "${NEXUS_REGISTRY}/2401198/travelstory-backend:v1"
        NODE_BASE      = "${NEXUS_REGISTRY}/library/node:18"
    }

    stages {

        /* ===============================
              FRONTEND BUILD
           =============================== */
        stage('Install + Build Frontend') {
            steps {
                container('node') {
                    dir('frontend') {
                        sh '''
                            npm ci
                            npm run build
                        '''
                    }
                }
            }
        }

        /* ===============================
              BACKEND BUILD
           =============================== */
        stage('Install Backend Packages') {
            steps {
                container('node') {
                    dir('backend') {
                        sh '''
                            npm ci
                        '''
                    }
                }
            }
        }

        /* ===============================
              DOCKER BUILD + PUSH (using Nexus)
           =============================== */
        stage('Build, Tag & Push Docker Images') {
            steps {
                container('dind') {
                    // Use credentials stored in Jenkins (username/password)
                    withCredentials([usernamePassword(credentialsId: 'nexus-docker-cred', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                        sh '''
                        set -euo pipefail
                        echo "=== Configure Docker daemon to allow insecure Nexus (HTTP) registry ==="
                        mkdir -p /etc/docker
                        cat > /etc/docker/daemon.json <<'EOF'
                        {
                          "insecure-registries": ["${NEXUS_REGISTRY}"]
                        }
EOF
                        # restart dockerd inside the dind container
                        pkill dockerd || true
                        nohup dockerd --host=unix:///var/run/docker.sock --storage-driver=overlay2 >/tmp/dockerd.log 2>&1 &
                        sleep 8

                        echo "=== Login to Nexus registry ==="
                        docker login ${NEXUS_REGISTRY} -u "$NEXUS_USER" -p "$NEXUS_PASS"

                        echo "=== Build frontend image from frontend/ (using NODE base from Nexus) ==="
                        docker build --build-arg NODE_IMAGE=${NODE_BASE} -t ${FRONTEND_IMAGE} frontend/

                        echo "=== Build backend image from backend/ (using NODE base from Nexus) ==="
                        docker build --build-arg NODE_IMAGE=${NODE_BASE} -t ${BACKEND_IMAGE} backend/

                        echo "=== Push frontend image ==="
                        docker push ${FRONTEND_IMAGE}

                        echo "=== Push backend image ==="
                        docker push ${BACKEND_IMAGE}
                        '''
                    }
                }
            }
        }

        /* ===============================
              SONARQUBE SCAN (secure token)
           =============================== */
        stage('SonarQube Analysis') {
            steps {
                container('sonar-scanner') {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        sh '''
                        sonar-scanner \
                          -Dsonar.projectKey=2401198_TravelStory \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=http://sonarqube.imcc.com \
                          -Dsonar.login=$SONAR_TOKEN
                        '''
                    }
                }
            }
        }

        /* ===============================
              KUBERNETES DEPLOYMENT
           =============================== */
        stage('Deploy to Kubernetes') {
            steps {
                container('kubectl') {
                    sh '''
                    set -euo pipefail

                    # ensure namespace exists
                    if ! kubectl get ns ${APP_NAMESPACE} >/dev/null 2>&1; then
                        kubectl create ns ${APP_NAMESPACE}
                    fi

                    # Update k8s manifests (ensure images point to Nexus images)
                    # It's recommended to update k8s/deployment.yaml to use the Nexus image paths.
                    # For safety, we patch deployments here to use the newly built images.
                    kubectl apply -f k8s/service.yaml -n ${APP_NAMESPACE} || true
                    kubectl apply -f k8s/deployment.yaml -n ${APP_NAMESPACE} || true

                    # Patch deployments (if deployments exist) to use the freshly pushed images
                    kubectl -n ${APP_NAMESPACE} set image deployment/travelstory-frontend-deployment travelstory-frontend=${FRONTEND_IMAGE} || true
                    kubectl -n ${APP_NAMESPACE} set image deployment/travelstory-backend-deployment travelstory-backend=${BACKEND_IMAGE} || true

                    # Wait for rollout — use actual deployment name in your k8s manifests, adjust if different
                    kubectl rollout status deployment/travelstory-frontend-deployment -n ${APP_NAMESPACE} --timeout=120s || true
                    kubectl rollout status deployment/travelstory-backend-deployment -n ${APP_NAMESPACE} --timeout=120s || true
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully."
        }
        failure {
            echo "Pipeline failed — check console log for details."
        }
    }
}
