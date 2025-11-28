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
      command: ['sh', '-c', 'sleep infinity']
      tty: true
      securityContext:
        runAsUser: 0
      env:
        - name: KUBECONFIG
          value: /kube/kubeconfig
      volumeMounts:
        - mountPath: /kube/kubeconfig
          name: kubeconfig-secret
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

        stage('Frontend Build') {
            steps {
                container('node') {
                    dir('frontend') {
                        sh '''
                            npm cache clean --force
                            rm -rf node_modules package-lock.json
                            npm install --legacy-peer-deps
                            npm install -g vite
                            npm run build
                        '''
                    }
                }
            }
        }

        stage('Backend Install') {
            steps {
                container('node') {
                    dir('backend') {
                        sh 'npm install'
                    }
                }
            }
        }

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

        stage('Login to Nexus') {
            steps {
                container('dind') {
                    sh '''
                        docker login nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085 \
                        -u student -p Imcc@2025
                    '''
                }
            }
        }

        stage('Push Docker Images') {
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

        stage('Create Image Pull Secret & App Secret') {
            steps {
                container('kubectl') {
                    sh '''
                        # Create image pull secret if it doesn't exist
                        kubectl create secret docker-registry nexus-credentials \
                        --docker-server=nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085 \
                        --docker-username=student \
                        --docker-password=Imcc@2025 \
                        --docker-email=student@example.com \
                        -n 2401198 --dry-run=client -o yaml | kubectl apply -f -

                        # Create app secret if it doesn't exist
                        kubectl create secret generic travelstory-secret \
                          -n 2401198 \
                          --from-literal=mongo_url="mongodb+srv://vaishnavitapasvi:alohomora@travelstory.7r8gs.mongodb.net/?retryWrites=true&w=majority&appName=travelstory" \
                          --from-literal=jwt_secret="1dd94577be1ccade99811a50934175c9a233e9fdaebf9b0c140e8473535a217f3bfeb85148f33319e3cfb412e9b887c0b388386f011c35d738d3df033706d64f" \
                          --dry-run=client -o yaml | kubectl apply -f -
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('kubectl') {
                    sh '''
                        kubectl get ns 2401198 || kubectl create ns 2401198
                        kubectl apply -f k8s/deployment.yaml -n 2401198
                        kubectl apply -f k8s/service.yaml -n 2401198
                        kubectl rollout status deployment/travelstory-deployment -n 2401198 --timeout=300s || echo "Rollout may be delayed"
                        kubectl get pods -n 2401198
                        kubectl get events -n 2401198 --sort-by=.metadata.creationTimestamp | tail -20
                    '''
                }
            }
        }
    }
}
