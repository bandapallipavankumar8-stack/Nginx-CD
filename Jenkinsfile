pipeline {
    agent any

    environment {
        S3_BUCKET = 's3://nginx-ci/packages'
        AWS_REGION = 'ap-south-1'
        
        // Replace with your real Mumbai EC2 Public IP address
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
                echo 'Listing workspace files for verification:'
                sh "ls -la"
                
                echo 'Packaging project files using Jenkins Native Utility...'
                // Native Jenkins zip step: safely packages files without CLI errors
                zip zipFile: "package-${BUILD_NUMBER}.zip", archive: false
            }
        }

        stage('Upload to S3') {
            steps {
                echo 'Uploading package to S3...'
                sh "aws s3 cp package-${BUILD_NUMBER}.zip ${env.S3_BUCKET}/package-${BUILD_NUMBER}.zip --region ${env.AWS_REGION}"
            }
        }

        stage('Configure Nginx & Deploy') {
            steps {
                echo "Connecting to EC2 Server at ${env.EC2_PUBLIC_IP}..."
                
                // Uses the SSH credentials ID configured in your Jenkins global settings
                sshagent(['ec2-mumbai-key']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${env.EC2_PUBLIC_IP} '
                        echo "==== Updating System & Installing Nginx ===="
                        sudo apt update -y && sudo apt install nginx awscli unzip -y
                        
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
                        
                        echo "==== Target Nginx Status ===="
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
