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

    stages {

        /* ===============================
              FRONTEND BUILD
           =============================== */
        stage('Install + Build Frontend') {
            steps {
                container('node') {
                    dir('frontend') {    // <-- FIXED HERE
                        sh '''
                            npm install
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
                    dir('backend') {      // <-- NEW
                        sh '''
                            npm install
                        '''
                    }
                }
            }
        }

        /* ===============================
              DOCKER BUILD
           =============================== */
        stage('Build Docker Image') {
            steps {
                container('dind') {
                    sh '''
                        sleep 10

                        # Build frontend image from frontend folder
                        docker build -t travelstory-frontend:latest frontend/

                        # Build backend image from backend folder
                        docker build -t travelstory-backend:latest backend/
                    '''
                }
            }
        }

        /* ===============================
              SONARQUBE SCAN
           =============================== */
        stage('SonarQube Analysis') {
            steps {
                container('sonar-scanner') {
                    sh '''
                        sonar-scanner \
                            -Dsonar.projectKey=2401198_TravelStory \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=http://sonarqube.imcc.com/ \
                            -Dsonar.login=sqp_fadcff67dcae94770ba78218d395a28ef52fefc4
                    '''
                }
            }
        }

        /* ===============================
              DOCKER LOGIN
           =============================== */
        stage('Login to Nexus Registry') {
            steps {
                container('dind') {
                    sh '''
                        docker login nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085 \
                        -u admin -p Changeme@2025
                    '''
                }
            }
        }

        /* ===============================
              PUSH IMAGES TO NEXUS
           =============================== */
        stage('Push to Nexus') {
            steps {
                container('dind') {
                    sh '''
                        docker tag travelstory-frontend:latest nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085/2401198/travelstory-frontend:v1
                        docker push nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085/2401198/travelstory-frontend:v1

                        docker tag travelstory-backend:latest nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085/2401198/travelstory-backend:v1
                        docker push nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085/2401198/travelstory-backend:v1
                    '''
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
                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml

                        kubectl rollout status deployment/recipe-finder-deployment -n 2401198
                    '''
                }
            }
        }
    }
}
