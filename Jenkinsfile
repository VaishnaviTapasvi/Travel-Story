pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:

  - name: node
    image: node:20
    command: ['cat']
    tty: true

  - name: sonar-scanner
    image: sonarsource/sonar-scanner-cli
    command: ['cat']
    tty: true

  - name: kubectl
    image: bitnami/kubectl:latest
    command:
      - /bin/sh
      - -c
      - sleep infinity
    tty: true
    securityContext:
      runAsUser: 0
      readOnlyRootFilesystem: false
    env:
      - name: KUBECONFIG
        value: /kube/config
    volumeMounts:
      - name: kubeconfig-secret
        mountPath: /kube/config
        subPath: kubeconfig

  - name: dind
    image: docker:dind
    args:
      - "--storage-driver=overlay2"
      - "--insecure-registry=nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085"
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

        /* -------------------------
           FRONTEND BUILD
        -------------------------- */
        stage('Install + Build Frontend') {
            steps {
                container('node') {
                    dir('frontend') {
                        sh '''
                            echo "Cleaning old npm cache..."
                            npm cache clean --force

                            echo "Removing node_modules & lock file..."
                            rm -rf node_modules package-lock.json

                            echo "Installing dependencies..."
                            npm install --legacy-peer-deps

                            echo "Installing Vite globally..."
                            npm install -g vite

                            echo "Running Vite build..."
                            npm run build
                        '''
                    }
                }
            }
        }

        /* -------------------------
           BACKEND INSTALL
        -------------------------- */
        stage('Install Backend Dependencies') {
            steps {
                container('node') {
                    dir('backend') {
                        sh '''
                            npm install
                        '''
                    }
                }
            }
        }

        /* -------------------------
           BUILD DOCKER IMAGES
        -------------------------- */
        stage('Build Docker Images') {
            steps {
                container('dind') {
                    sh '''
                        sleep 10
                        docker build -t travelstory-frontend:latest ./frontend
                        docker build -t travelstory-backend:latest ./backend
                    '''
                }
            }
        }

        /* -------------------------
           SONARQUBE ANALYSIS
        -------------------------- */
        stage('SonarQube Analysis') {
            steps {
                container('sonar-scanner') {
                    dir('backend') {
                        sh '''
                            sonar-scanner \
                                -Dsonar.projectKey=2401198_TravelStory \
                                -Dsonar.sources=. \
                                -Dsonar.host.url=http://my-sonarqube-sonarqube.sonarqube.svc.cluster.local:9000 \
                                -Dsonar.login=sqp_fadcff67dcae94770ba78218d395a28ef52fefc4
                        '''
                    }
                }
            }
        }

        /* -------------------------
           LOGIN TO NEXUS
        -------------------------- */
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

        /* -------------------------
           PUSH IMAGES TO NEXUS
        -------------------------- */
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

        /* -------------------------
           KUBERNETES DEPLOYMENT
        -------------------------- */
        stage('Deploy to Kubernetes') {
    steps {
        container('kubectl') {
            sh '''
                echo "Ensuring namespace exists..."
                kubectl get ns 2401198 || kubectl create ns 2401198

                echo "Applying deployment and service..."
                kubectl apply -f k8s/deployment.yaml -n 2401198
                kubectl apply -f k8s/service.yaml -n 2401198

                echo "Checking rollout status..."
                kubectl rollout status deployment/travelstory-deployment -n 2401198
            '''
        }
    }
}

    }
}
