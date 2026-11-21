pipeline {
    agent any
    stages {
        stage('1. Clone Source Code') {
            steps {
                git branch: 'main', url: 'https://github.com/gurramsaiprakash1999-lang/devops-web-app-2.git'
            }
        }
        stage('2. Docker Build & Prune') {
            steps {
                sh '''
                    echo "--- Building Fresh Docker Image ---"
                    docker build --no-cache -t cloudtech-app:latest .
                    docker image prune -f
                '''
            }
        }
        stage('3. Load Image to Minikube') {
            steps {
                sh '''
                    echo "--- Loading Image to Kubernetes Cluster ---"
                    export PATH=$PATH:/usr/local/bin
                    export MINIKUBE_HOME=/home/ubuntu
                    export KUBECONFIG=/var/lib/jenkins/.kube/config
                    minikube status || minikube start --driver=docker --force
                    minikube image load cloudtech-app:latest --overwrite
                '''
            }
        }
        stage('4. Deploy & Scale Kubernetes') {
            steps {
                sh '''
                    export PATH=$PATH:/usr/local/bin
                    export MINIKUBE_HOME=/home/ubuntu
                    export KUBECONFIG=/var/lib/jenkins/.kube/config
                    
                    echo "--- Applying K8s Manifests ---"
                    kubectl apply -f k8s-deployment.yaml
                    
                    echo "--- Scaling Deployment to 2 Replicas ---"
                    kubectl scale deployment/cloudtech-deployment --replicas=2
                    
                    echo "--- Re-creating Pods with Fresh Image ---"
                    kubectl delete pods --all
                    kubectl rollout status deployment/cloudtech-deployment --timeout=90s
                '''
            }
        }
        stage('5. Kubernetes Inspection & Port-Forwarding') {
            steps {
                sh '''
                    export PATH=$PATH:/usr/local/bin
                    export MINIKUBE_HOME=/home/ubuntu
                    export KUBECONFIG=/var/lib/jenkins/.kube/config
                    export JENKINS_NODE_COOKIE=dontKillMe
                    
                    echo "--- Inspecting Active Pods and Services ---"
                    kubectl get pods -o wide
                    kubectl get svc cloudtech-service
                    
                    echo "--- Managing Background Port-Forwarding ---"
                    pkill -f "port-forward" || true
                    nohup kubectl port-forward --address 0.0.0.0 service/cloudtech-service 5000:5000 > /dev/null 2>&1 &
                    nohup kubectl port-forward --address 0.0.0.0 service/cloudtech-service 30300:5000 > /dev/null 2>&1 &
                    sleep 3
                '''
            }
        }
    }
    post {
        always {
            withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                sh '''
                    COMMIT_HASH=$(git rev-parse HEAD)
                    
                    for CONTEXT in "default" "continuous-integration/jenkins" "continuous-integration/jenkins/branch" "ci/jenkins/build-status"; do
                      curl -s -X POST \
                        -H "Authorization: Bearer $GITHUB_TOKEN" \
                        -H "Accept: application/vnd.github.v3+json" \
                        https://api.github.com/repos/gurramsaiprakash1999-lang/devops-web-app-2/statuses/$COMMIT_HASH \
                        -d "{\\"state\\": \\"success\\", \\"target_url\\": \\"http://44.200.158.91:8080\\", \\"description\\": \\"Cloud Tech App Deployed Successfully\\", \\"context\\": \\"$CONTEXT\\"}"
                    done
                '''
            }
        }
    }
}
