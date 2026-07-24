pipeline {
    agent any

    parameters {
        string(name: 'CI_BUILD_NUMBER', defaultValue: '1', description: 'Passed from Nginx-CI')
    }

    environment {
        // FIXED: This acts as your base path folder to dynamically fetch any build number package safely
        S3_PUBLIC_URL = 'https://nginx-ci.s3.ap-south-1.amazonaws.com/packages'
        EC2_PUBLIC_IP = '65.0.110.234' 
    }

    stages {
        stage('Configure VM & Deploy') {
            steps {
                echo "Connecting to Amazon Linux EC2 to deploy package-${params.CI_BUILD_NUMBER}.zip..."
                
                withCredentials([sshUserPrivateKey(credentialsId: 'ec2-mumbai-key', keyFileVariable: 'TEMP_KEY')]) {
                    sh """
                    ssh -o StrictHostKeyChecking=no -i \${TEMP_KEY} ec2-user@${env.EC2_PUBLIC_IP} "
                        echo '==== Installing Dependencies ===='
                        sudo yum install git nginx unzip -y
                        
                        sudo systemctl start nginx
                        sudo systemctl enable nginx
                        
                        echo '==== Cleaning Old Web Files ===='
                        sudo rm -rf /usr/share/nginx/html/*
                        
                        echo '==== Fetching Package via Public HTTP URL ===='
                        # DYNAMIC EXECUTOR: This automatically injects the current build number onto your S3 URL
                        curl -sL ${env.S3_PUBLIC_URL}/package-${params.CI_BUILD_NUMBER}.zip -o /home/ec2-user/package.zip
                        
                        echo '==== Deploying New Web Content ===='
                        sudo unzip -o /home/ec2-user/package.zip -d /usr/share/nginx/html/
                        
                        echo '==== Bypassing 403 Forbidden: Fixing Folder Permissions ===='
                        sudo chown -R nginx:nginx /usr/share/nginx/html
                        sudo chmod -R 755 /usr/share/nginx/html
                        sudo chmod 755 /usr/share/nginx /usr/share /usr
                        sudo chcon -Rt httpd_sys_content_t /usr/share/nginx/html
                        
                        rm -f /home/ec2-user/package.zip
                        sudo systemctl reload nginx
                        echo '==== DEPLOYMENT COMPLETE ===='
                    "
                    """
                }
            }
        }
    }
}
