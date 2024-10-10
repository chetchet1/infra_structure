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

        // Apply Storage Class
        stage('Apply Storage Class') {
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

        // Apply Kafka PVC
        stage('Apply Kafka PVC') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-key']]) {
                        bat '''
                        kubectl apply -f E:/docker_Logi/infra_structure/kafka-pvc.yaml
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
                        
                        REM Install Kafka (with corrected settings)
                        helm install kafka bitnami/kafka -f E:/docker_Logi/infra_structure/values.yaml

                        REM Wait for the Kafka pod to be in the Running state
                        echo Waiting for Kafka pods to be in Running state...
                        :loop
                        kubectl get pods -l app.kubernetes.io/name=kafka -o jsonpath="{.items[?(@.status.phase=='Running')].metadata.name}" | findstr /C:"kafka-controller" >nul
                        if errorlevel 1 (
                            timeout /t 10 >nul
                            goto loop
                        )
                        echo Kafka pods are now Running!
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
		def kafka_pod = powershell(script: 'kubectl get pod -l app.kubernetes.io/name=kafka -o jsonpath="{.items[0].metadata.name}"', returnStdout: true).trim()                        

		echo "kafka_pod: ${kafka_pod}"

		 // 3. Kafka Topics 생성
                	bat """
                    	   kubectl get pods -l app.kubernetes.io/name=kafka
                    	   kubectl get storageclass
                    
                    	   kubectl exec -it ${kafka_pod} -- bash -c "kafka-topics.sh --create --topic test-topic --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1"
                	"""
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo 'Infrastructure successfully deployed, PVC created, and topics created!'
        }
        failure {
            echo 'Deployment failed.'
        }
    }
}
