pipeline {
    agent any
    
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
                bat '''
                aws eks update-kubeconfig --region ap-northeast-2 --name test-eks-cluster
                eksctl create addon --name aws-ebs-csi-driver --cluster test-eks-cluster --service-account-role-arn arn:aws:iam::339713037008:role/AmazonEKSEBSCSIRole --force
                '''
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
    }
    
    post {
        success {
            echo 'Infrastructure successfully deployed!'
        }
        failure {
            echo 'Deployment failed.'
        }
    }
}
