pipeline {
    agent any

    parameters {
        string(name: 'CI_BUILD_NUMBER', defaultValue: '1', description: 'Passed from Nginx-CI')
    }

    environment {
        // Target app server IP address
        EC2_PUBLIC_IP = '3.109.207.176' 
    }

    stages {
        stage('Configure VM & Deploy') {
            steps {
                echo "Connecting to Amazon Linux EC2 to deploy package-${params.CI_BUILD_NUMBER}.zip..."
                
                withCredentials([sshUserPrivateKey(credentialsId: 'ec2-mumbai-key', keyFileVariable: 'TEMP_KEY')]) {
                    // Double quotes force Jenkins to bake the right URL straight into the command
                    sh """
                    ssh -o StrictHostKeyChecking=no -T -i \${TEMP_KEY} ec2-user@${env.EC2_PUBLIC_IP} "
                        echo '==== Cleaning Package Cache ===='
                        sudo yum clean all
                        
                        echo '==== Installing Dependencies Safely ===='
                        sudo yum install git nginx unzip -y -q
                        
                        sudo systemctl start nginx
                        sudo systemctl enable nginx
                        
                        echo '==== Cleaning Old Web Files ===='
                        sudo rm -rf /usr/share/nginx/html/*
                        
                        echo '==== Fetching Package via Public HTTP URL ===='
                        # FORCE DEPLOY FIX: Full explicit URL with proper parameter matching
                        if ! curl -sfL https://amazonaws.com{params.CI_BUILD_NUMBER}.zip -o /home/ec2-user/package.zip; then
                            echo 'ERROR: Failed to download package from S3. Please verify the build number or S3 permissions.'
                            exit 1
                        fi
                        
                        echo '==== Validating Zip File Integrity ===='
                        if ! unzip -t /home/ec2-user/package.zip >/dev/null 2>&1; then
                            echo 'ERROR: Downloaded file is corrupt or is an AWS XML error page.'
                            rm -f /home/ec2-user/package.zip
                            exit 1
                        fi
                        
                        echo '==== Deploying New Web Content ===='
                        sudo unzip -o /home/ec2-user/package.zip -d /usr/share/nginx/html/
                        
                        echo '==== Bypassing 403 Forbidden: Fixing Folder Permissions ===='
                        sudo chown -R nginx:nginx /usr/share/nginx/html
                        sudo chmod -R 755 /usr/share/nginx/html
                        sudo chmod 755 /usr/share/nginx /usr/share /usr
                        
                        if command -v chcon >/dev/null 2>&1; then
                            sudo chcon -Rt httpd_sys_content_t /usr/share/nginx/html || true
                        fi
                        
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
