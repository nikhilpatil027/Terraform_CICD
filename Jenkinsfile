pipeline {
    agent any

    parameters {
        string(name: 'environment', defaultValue: 'terraform', description: 'Workspace/environment file to use for deployment')
        booleanParam(name: 'autoApprove', defaultValue: false, description: 'Automatically run apply after generating plan?')
        booleanParam(name: 'destroy', defaultValue: false, description: 'Destroy Terraform build?')

    }


     environment {
        AWS_ACCESS_KEY_ID     = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
        AWS_DEFAULT_REGION    = "ap-south-1"
        SSH_KEY = "/home/ubuntu/.ssh/Terraform.pem"  // Replace with your private key
        ANSIBLE_PLAYBOOK = "webserver.yml"
        GIT_REPO = "https://github.com/nikhilpatil027/Ansible_Playbook.git"
    }


    stages {
        stage('checkout') {
            when {
                not {
                    equals expected: true, actual: params.destroy
                }
            }
            steps {
                    script {
                        dir("terraform") {
                            
                            sh '''
                                if [ -d "Terraform_CICD" ]; then
                                    echo "Directory exists. Deleting it..."
                                    rm -rf Terraform_CICD
                                fi
                                echo "Cloning the repository..."
                                git clone "https://github.com/nikhilpatil027/Terraform_CICD.git"
                            '''
                        }
                    }
                }
            }

        stage('Plan') {
            when {
                not {
                    equals expected: true, actual: params.destroy
                }
            }
            
          steps {
                dir('terraform/Terraform_CICD') {  // Adjust path as needed
                    sh 'terraform init -input=false'
                    sh 'terraform workspace select ${environment} || terraform workspace new ${environment}'
                    sh "terraform plan -input=false -out tfplan "
                    sh 'terraform show -no-color tfplan > tfplan.txt'
                }
            }
        }
        stage('Approval') {
           when {
               not {
                   equals expected: true, actual: params.autoApprove
               }
               not {
                    equals expected: true, actual: params.destroy
                }
           }
           steps {
               script {
                    def plan = readFile 'tfplan.txt'
                    input message: "Do you want to apply the plan?",
                    parameters: [text(name: 'Plan', description: 'Please review the plan', defaultValue: plan)]
               }
           }
       }

        stage('Apply') {
            when {
                not {
                    equals expected: true, actual: params.destroy
                }
            }
            
           steps {
                dir('terraform/Terraform_CICD') {  // Add this directory block, same as in Plan stage
                    sh "terraform apply -input=false tfplan"
                }
            }
        }
        
        stage('Destroy') {
            when {
                equals expected: true, actual: params.destroy
            }
        
        steps {
                dir('terraform/Terraform_CICD') {  // Add this directory block, same as in Plan stage
           sh "terraform destroy --auto-approve"
                }
        }
    }

         stage('Checkout Code') {
            steps {
                git branch: 'main', url: "${GIT_REPO}"
            }
        }

          stage('Get EC2 Public IP') {
            steps {
                script {
                    dir('terraform/Terraform_CICD') {
                    def output = sh(script: "terraform output -raw ec2_public_ip", returnStdout: true).trim()
                    env.EC2_IP = output
                    echo "EC2 Public IP: ${env.EC2_IP}"
                }
                }
            }
        }

       stage('Run Ansible Playbook from Local') {
        steps {
            withCredentials([sshUserPrivateKey(credentialsId: 'ansible-key', keyFileVariable: 'SSH_KEY')]) {
                sh '''
                    export ANSIBLE_HOST_KEY_CHECKING=False
                    ansible-playbook -i "${EC2_IP}," --private-key "${SSH_KEY}" -u ubuntu webserver.yml
                '''
            }
        }
    }


  }
}

