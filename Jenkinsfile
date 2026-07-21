pipeline {
    agent any

    parameters {
        string(name: 'CI_BUILD_NUMBER', defaultValue: '1', description: 'Passed from CI pipeline')
    }

    environment {
        S3_BUCKET = 's3://nginx-ci/packages'
        AWS_REGION = 'ap-south-1'
        // FIXED: Updated to your current active EC2 Public IP address
        EC2_PUBLIC_IP = '13.203.218.120' 
    }

    stages {
        stage('Configure VM & Deploy') {
            steps {
                echo "Connecting to Amazon Linux EC2 to deploy package-${params.CI_BUILD_NUMBER}.zip..."
                
                // Ensure 'ec2-mumbai-key' matches your exact ID under Manage Jenkins -> Credentials
                withCredentials([sshUserPrivateKey(credentialsId: 'ec2-mumbai-key', keyFileVariable: 'TEMP_KEY')]) {
                    // FIXED: Changed to triple double-quotes and escaped interior variables with a backslash
                    sh """
                    ssh -o StrictHostKeyChecking=no -i \${TEMP_KEY} ec2-user@${env.EC2_PUBLIC_IP} "
                        echo '==== Installing Dependencies ===='
                        sudo yum install git nginx unzip -y
                        
                        sudo systemctl start nginx
                        sudo systemctl enable nginx
                        
                        echo '==== Cleaning Old Web Files ===='
                        sudo rm -rf /usr/share/nginx/html/*
                        
                        echo '==== Fetching Package from S3 ===='
                        aws s3 cp ${env.S3_BUCKET}/package-${params.CI_BUILD_NUMBER}.zip /home/ec2-user/package.zip --region ${env.AWS_REGION}
                        
                        echo '==== Deploying New Web Content ===='
                        sudo unzip /home/ec2-user/package.zip -d /usr/share/nginx/html/
                        
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
