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
        
        // EBS CSI 드라이버 설치
        stage('Install EBS CSI Driver') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-key']]) {
                        // KUBECONFIG 환경변수로 kubeconfig 파일을 설정하여 eksctl 명령 실행
                        bat '''
                        set KUBECONFIG=C:\\Windows\\system32\\config\\systemprofile\\.kube\\config
                        aws eks update-kubeconfig --region ap-northeast-2 --name test-eks-cluster
                        eksctl utils associate-iam-oidc-provider --region=ap-northeast-2 --cluster=test-eks-cluster --approve
                        eksctl get cluster --region ap-northeast-2
                        type C:\\Windows\\system32\\config\\systemprofile\\.kube\\config
                        kubectl get nodes
                        aws eks describe-cluster --name test-eks-cluster --region ap-northeast-2 --query "cluster.identity.oidc.issuer" --output text
                        
                        kubectl apply -f E:/docker_Logi/infra_structure/ebs-csi-service-account.yaml
                        
                        eksctl create addon --name aws-ebs-csi-driver --cluster test-eks-cluster --service-account-role-arn arn:aws:iam::339713037008:role/AmazonEKSEBSCSIRole --force
                        '''
                    }
                }
            }
        }

        // Kubernetes 스토리지 클래스 및 PVC 생성
        stage('Apply Storage Class and PVCs') {
            steps {
                bat '''
                kubectl apply -f E:/docker_Logi/infra_structure/storage-class.yaml
                kubectl apply -f E:/docker_Logi/infra_structure/zookeeper-pvc.yaml
                kubectl apply -f E:/docker_Logi/infra_structure/kafka-pvc.yaml
                '''
            }
        }

        // Zookeeper 및 Kafka 배포
        stage('Deploy Zookeeper and Kafka') {
            steps {
                bat '''
                kubectl apply -f E:/docker_Logi/infra_structure/zookeeper-deployment.yaml
                kubectl apply -f E:/docker_Logi/infra_structure/kafka-deployment.yaml
                kubectl apply -f E:/docker_Logi/infra_structure/zookeeper-service.yaml
                kubectl apply -f E:/docker_Logi/infra_structure/kafka-service.yaml
                '''
            }
        }

        // Kafka 토픽 생성
        stage('Create Kafka Topics') {
            steps {
                bat '''
                kubectl exec -it $(kubectl get pod -l app=kafka -o jsonpath='{.items[0].metadata.name}') -- bash -c "kafka-topics.sh --create --topic test-topic --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1"
                '''
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
