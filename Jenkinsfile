pipeline {
    agent any

    environment {
        // Target AWS S3 Bucket Name
        S3_BUCKET = 's3://nginx-ci/packages'
        AWS_REGION = 'ap-south-1'
        
        // Target Mumbai EC2 Public IP address
        EC2_PUBLIC_IP = '13.203.158.156' 
    }

    stages {
        stage('Checkout Source') {
            steps {
                checkout scm
            }
        }

        stage('Package Code') {
            steps {
                echo 'Listing workspace files to verify index.html presence:'
                sh "ls -la"
                
                echo 'Packaging index.html using the standard Linux zip client...'
                sh "zip -r package-${BUILD_NUMBER}.zip index.html"
            }
        }

        stage('Upload to S3') {
            steps {
                echo 'Uploading package to S3...'
                sh "aws s3 cp package-${BUILD_NUMBER}.zip ${env.S3_BUCKET}/package-${BUILD_NUMBER}.zip --region ${env.AWS_REGION}"
            }
        }

        stage('Configure VM & Deploy') {
            steps {
                echo "Connecting to EC2 Server at ${env.EC2_PUBLIC_IP}..."
                
                // Uses the SSH credentials ID configured in your Jenkins global settings
                sshagent(['ec2-mumbai-key']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${env.EC2_PUBLIC_IP} '
                        echo "==== Updating System Packages ===="
                        sudo apt update -y
                        
                        echo "==== Step 1: Installing Git on Target VM ===="
                        sudo apt install git -y
                        
                        echo "==== Step 2: Installing Nginx, AWS CLI, and Unzip ===="
                        sudo apt install nginx awscli unzip -y
                        
                        echo "==== Starting Nginx Service ===="
                        sudo systemctl start nginx
                        sudo systemctl enable nginx
                        
                        echo "==== Cleaning Old Web Files ===="
                        sudo rm -rf /var/www/html/*
                        
                        echo "==== Fetching Package from S3 ===="
                        aws s3 cp ${env.S3_BUCKET}/package-${BUILD_NUMBER}.zip ./package.zip --region ${env.AWS_REGION}
                        
                        echo "==== Deploying New Web Content ===="
                        sudo unzip package.zip -d /var/www/html/
                        
                        echo "==== Post-Deployment Clean up ===="
                        rm package.zip
                        
                        echo "==== Verification ===="
                        git --version
                        sudo systemctl is-active nginx
                    '
                    """
                }
            }
        }
    }

    post {
        always {
            sh "rm -f package-${BUILD_NUMBER}.zip"
        }
    }
}
