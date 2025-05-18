pipeline {
    agent any

    environment {
        ARM_CLIENT_ID       = credentials('AZURE_CLIENT_ID')
        ARM_CLIENT_SECRET   = credentials('AZURE_CLIENT_SECRET')        
        ARM_SUBSCRIPTION_ID= credentials('AZURE_SUBSCRIPTION_ID')
        ARM_TENANT_ID= credentials('AZURE_TENANT_ID')
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Checking out source code..."
                git branch: 'master', 
                url: 'https://github.com/neirezcher/terraform-jenkins.git'
            }
        }

        stage('Terraform Init') {
            steps {
                echo "Initializing Terraform..."
                 sh '''
                 
                    terraform init
                    '''
            }
        }

        stage('Terraform Plan') {
            steps {
                echo "Generating Terraform plan..."
               
                        sh '''
                        export ARM_USE_OIDC=true
                        terraform plan 
                            -out=tfplan
                        '''
                        archiveArtifacts artifacts: 'tfplan'
                    
            }
        }

        stage('Terraform Apply') {
            steps {
                sh '''
                    terraform apply -auto-approve tfplan
                    '''
            }
        }
        stage('Get VM IP') {
            steps {
                script {
                    // Store IP in environment variable
                    env.VM_IP = sh(
                        script: 'cd terraform && terraform output -raw azurerm_public_ip', 
                        returnStdout: true
                    ).trim()
                    echo "VM IP: ${env.VM_IP}"
                }
            }
        }
        stage('Run Ansible') {
            steps{
                withCredentials([sshUserPrivateKey(
                        credentialsId: 'ANSIBLE_SSH_PRIVATE_KEY',
                        keyFileVariable: 'SSH_KEY'
                    )]) {
                        sh """
                        # Verify playbook exists
                        if [ ! -f playbook.yml ]; then
                            echo "ERROR: playbook.yml not found in \$(pwd)"
                            ls -l
                            exit 1
                        fi
                        
                        # Test connection
                        ssh -o StrictHostKeyChecking=no -i $SSH_KEY jenkinsadmin@${env.VM_IP} 'sudo whoami' || {
                            echo "ERROR: SSH failed"
                            exit 1
                        }
                        
                        # Run Ansible with full path
                        ansible-playbook -i '${env.VM_IP},' \$(pwd)/playbook.yml -vvv \\
                            --private-key=$SSH_KEY \\
                            --user=jenkinsadmin \\
                            --become \\
                            -e "ansible_become_pass=''" \\
                            -e "ansible_ssh_common_args='-o StrictHostKeyChecking=no -o ConnectTimeout=30'"
                        """
                    }
            }

        }
    }
     post {
        always {
            echo "Cleaning up workspace..."
            cleanWs()
        }
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}
