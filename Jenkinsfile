pipeline {
    agent any

    environment {
        AWS_REGION = "region"
        AWS_ACCOUNT_ID = "aws account number"
        AWS_ACCESS_KEY_ID = "replaceaccesskey"
        AWS_SECRET_ACCESS_KEY = "replacesecretkey"
        ECR_REPO_BACKEND = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/kumaresan/backend"
        ECR_REPO_FRONTEND = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/kumaresan/frontend"
        KUBE_CONFIG = "${WORKSPACE}/kubeconfig"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/kumar-DevOps/Orchestration.git'
            }
        }

        stage('Configure AWS CLI') {
            steps {
                sh '''
                aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                aws configure set default.region $AWS_REGION
                '''
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                aws ecr get-login-password --region $AWS_REGION \
                | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                '''
            }
        }

        stage('Build Frontend Image') {
            steps {
                dir('frontend') {
                    sh '''
                    docker build -t frontend:latest .
                    docker tag frontend:latest $ECR_REPO_FRONTEND:latest
                    '''
                }
            }
        }

        stage('Build Backend Images') {
            parallel {
                stage('Streaming Service') {
                    steps {
                        dir('backend/streamingService') {
                            sh '''
                            docker build -t backend:streaming .
                            docker tag backend:streaming $ECR_REPO_BACKEND:streaming
                            '''
                        }
                    }
                }
                stage('Chat Service') {
                    steps {
                        dir('backend/chatService') {
                            sh '''
                            docker build -t backend:chat .
                            docker tag backend:chat $ECR_REPO_BACKEND:chat
                            '''
                        }
                    }
                }
                stage('Admin Service') {
                    steps {
                        dir('backend/adminService') {
                            sh '''
                            docker build -t backend:admin .
                            docker tag backend:admin $ECR_REPO_BACKEND:admin
                            '''
                        }
                    }
                }
                stage('Auth Service') {
                    steps {
                        dir('backend/authService') {
                            sh '''
                            docker build -t backend:auth .
                            docker tag backend:auth $ECR_REPO_BACKEND:auth
                            '''
                        }
                    }
                }
            }
        }

        stage('Push Images to ECR') {
            steps {
                sh '''
                docker push $ECR_REPO_FRONTEND:latest
                docker push $ECR_REPO_BACKEND:streaming
                docker push $ECR_REPO_BACKEND:chat
                docker push $ECR_REPO_BACKEND:admin
                docker push $ECR_REPO_BACKEND:auth
                '''
            }
        }

        stage('Create or Reuse EKS Cluster') {
            steps {
                sh '''
                # Download eksctl binary if not already present
                if [ ! -f ./eksctl ]; then
                  curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
                  mv /tmp/eksctl ./eksctl
                  chmod +x ./eksctl
                fi

                # Check if cluster already exists
                if aws eks describe-cluster --name streaming-cluster --region $AWS_REGION > /dev/null 2>&1; then
                  echo "Cluster 'streaming-cluster' already exists. Reusing it."
                else
                  echo "Cluster 'streaming-cluster' not found. Creating new cluster..."
                  ./eksctl create cluster \
                    --name streaming-cluster \
                    --region $AWS_REGION \
                    --nodes 3 \
                    --node-type t3.medium \
                    --managed
                fi

                # Always update kubeconfig to point to the cluster
                aws eks update-kubeconfig --region $AWS_REGION --name streaming-cluster --kubeconfig $KUBE_CONFIG
                '''
            }
        }

        stage('Install Helm & Kubectl Locally') {
            steps {
                sh '''
                # Install Helm
                curl -L https://get.helm.sh/helm-v3.14.0-linux-amd64.tar.gz -o helm.tar.gz
                tar -zxvf helm.tar.gz
                mv linux-amd64/helm ./helm
                chmod +x ./helm

                # Install kubectl
                curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                chmod +x kubectl
                '''
            }
        }

        stage('Sanity Check Cluster') {
            steps {
                sh '''
                export KUBECONFIG=$KUBE_CONFIG
                ./kubectl get nodes
                '''
            }
        }

        stage('Deploy to EKS via Helm') {
            steps {
                sh '''
                export KUBECONFIG=$KUBE_CONFIG
                ./helm upgrade --install streaming-app ./helm-chart \
                  --set frontend.image=$ECR_REPO_FRONTEND:latest \
                  --set backend.streaming.image=$ECR_REPO_BACKEND:streaming \
                  --set backend.chat.image=$ECR_REPO_BACKEND:chat \
                  --set backend.admin.image=$ECR_REPO_BACKEND:admin \
                  --set backend.auth.image=$ECR_REPO_BACKEND:auth
                '''
            }
        }

        stage('Enable Container Insights') {
            steps {
                sh '''
                aws eks update-cluster-config \
                  --region $AWS_REGION \
                  --name streaming-cluster \
                  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'
                '''
            }
        }

        stage('Enable Logging with Fluent Bit') {
            steps {
                sh '''
                export KUBECONFIG=$KUBE_CONFIG
                ./kubectl apply -f helm-chart/logging/fluent-bit.yaml
                '''
            }
        }
    }

    post {
        success {
            echo "✅ CI/CD pipeline completed successfully with monitoring & logging!"
        }
        failure {
            echo "❌ Pipeline failed. Check logs."
        }
        always {
            script {
                // Optional cleanup stage
                def cleanupCluster = false  // set to false if you want to keep cluster persistent
                if (cleanupCluster) {
                    sh '''
                    echo "Cleaning up EKS cluster..."
                    ./eksctl delete cluster --name streaming-cluster --region $AWS_REGION
                    '''
                } else {
                    echo "Cluster retained for reuse in future runs."
                }
            }
        }
    }
}
