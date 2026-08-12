pipeline {
    agent any

    parameters {
        string(name: 'CI_BUILD_NUMBER', defaultValue: '1', description: 'Passed from Nginx-CI')
    }

    environment {
        // Your target application machine Public IP address
        EC2_PUBLIC_IP = '3.109.207.176' 
        
        // Your exact S3 Bucket storage path
        S3_BUCKET_PATH = 's3://nginx-ci/packages'
    }

    stages {
        stage('Fetch from S3 & Deploy to VM') {
            steps {
                echo "Fetching package-${params.CI_BUILD_NUMBER}.zip on Jenkins Master..."
                
                // STEP 1: Download from S3 onto Jenkins Master (Works because Jenkins has the Admin IAM Role)
                sh "aws s3 cp ${env.S3_BUCKET_PATH}/package-${params.CI_BUILD_NUMBER}.zip ./package.zip --region ap-south-1"
                
                withCredentials([sshUserPrivateKey(credentialsId: 'ec2-mumbai-key', keyFileVariable: 'TEMP_KEY')]) {
                    
                    echo "Sending package to Target EC2 Machine..."
                    // STEP 2: Securely copy (SCP) the file from Jenkins onto your Target Server
                    sh "scp -o StrictHostKeyChecking=no -i \${TEMP_KEY} ./package.zip ec2-user@${env.EC2_PUBLIC_IP}:/home/ec2-user/package.zip"
                    
                    echo "Connecting via SSH to extract and deploy web files..."
                    // STEP 3: Connect via SSH to unpack the file and configure Nginx permissions
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
                        
                        echo '==== Validating Zip File Integrity ===='
                        if ! unzip -t /home/ec2-user/package.zip >/dev/null 2>&1; then
                            echo 'ERROR: Downloaded file is corrupt.'
                            rm -f /home/ec2-user/package.zip
                            exit 1
                        fi
                        
                        echo '==== Deploying New Web Content ===='
                        sudo unzip -o /home/ec2-user/package.zip -d /usr/share/nginx/html/
                        
                        echo '==== Fixing Folder Permissions ===='
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
    
    post {
        always {
            echo "Cleaning up workspace zip files on Jenkins Master..."
            // Cleans up the workspace on the Jenkins machine side
            sh "rm -f ./package.zip"
        }
    }
}
