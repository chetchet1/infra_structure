pipeline {
    agent any
    
    environment {
        KUBECONFIG = "C:\\Windows\\system32\\config\\systemprofile\\.kube\\config"
        REGION = 'ap-northeast-2'
        AWS_CREDENTIAL_NAME = 'aws-key'
    }

    stages {
        // Terraform으로 EKS 클러스터 및 자원 생성
        stage('Terraform Apply') {
            steps {
                dir('E:/docker_dev/terraform-codes') {
                    script {
                        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-key']]) {
                            bat '''
                            set AWS_ACCESS_KEY_ID=%AWS_ACCESS_KEY_ID%
                            set AWS_SECRET_ACCESS_KEY=%AWS_SECRET_ACCESS_KEY%
                            terraform apply -auto-approve
                            '''
                        }
                    }
                }
            }
        }
        
        // Install EBS CSI Driver
        stage('Install EBS CSI Driver') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-key']]) {
                        // Set KUBECONFIG environment variable to execute eksctl commands
                        bat '''
                        set KUBECONFIG=C:\\Windows\\system32\\config\\systemprofile\\.kube\\config
                        aws eks update-kubeconfig --region ap-northeast-2 --name test-eks-cluster
                        eksctl utils associate-iam-oidc-provider --region=ap-northeast-2 --cluster=test-eks-cluster --approve
                        eksctl get cluster --region ap-northeast-2
                        type C:\\Windows\\system32\\config\\systemprofile\\.kube\\config
                        kubectl get nodes
                        aws eks describe-cluster --name test-eks-cluster --region ap-northeast-2 --query "cluster.identity.oidc.issuer" --output text
                        
                        kubectl apply -f E:/docker_Logi/infra_structure/ebs-csi-service-account.yaml
                        '''
                    }
                }
            }
        }

        // Apply Storage Class and PVCs
        stage('Apply Storage Class and PVCs') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-key']]) {
                        bat '''
                        kubectl apply -f E:/docker_Logi/infra_structure/storage-class.yaml
                        '''
                    }
                }
            }
        }

        // Uninstall previous Zookeeper if it exists
        stage('Uninstall Previous Zookeeper') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-key']]) {
                        bat '''
                        set KUBECONFIG=C:\\Windows\\system32\\config\\systemprofile\\.kube\\config
                        helm uninstall kafka || echo "Kafka not installed or already uninstalled"
                        helm uninstall zookeeper || echo "Zookeeper not installed or already uninstalled"
                        '''
                    }
                }
            }
        }

        // Install Helm Chart for Zookeeper and Kafka
        stage('Install Zookeeper and Kafka with Helm') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-key']]) {
                        bat '''
                        set KUBECONFIG=C:\\Windows\\system32\\config\\systemprofile\\.kube\\config
                        helm repo add bitnami https://charts.bitnami.com/bitnami
                        helm repo update

                        REM Install Zookeeper
                        helm install zookeeper bitnami/zookeeper --set persistence.storageClass=ebs-sc --set persistence.size=8Gi
                        
                        REM Install Kafka (with corrected settings)
                        helm install kafka bitnami/kafka --set persistence.storageClass=ebs-sc --set persistence.size=8Gi --set zookeeper.enabled=true --set controller.enabled=false
                        '''
                    }
                }
            }
        }

        // Create Kafka Topics
        stage('Create Kafka Topics') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-key']]) {
                        bat '''
                        kubectl get pods -l app.kubernetes.io/name=kafka
                        FOR /F "tokens=*" %%i IN ('kubectl get pod -l app.kubernetes.io/name=kafka -o jsonpath="{.items[0].metadata.name}"') DO (
                            set KAFKA_POD=%%i
                        )

                        REM Check if KAFKA_POD is set correctly
                        echo Kafka Pod: %KAFKA_POD%

                        kubectl exec -it %KAFKA_POD% -- bash -c "kafka-topics.sh --create --topic test-topic --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1"
                        '''
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo 'Infrastructure successfully deployed and topics created!'
        }
        failure {
            echo 'Deployment failed.'
        }
    }
}
