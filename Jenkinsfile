pipeline {
    agent any

    environment {
        // Target AWS Configurations
        S3_BUCKET = 's3://nginx-ci/packages'
        AWS_REGION = 'ap-south-1'
        
        // Target Amazon Linux EC2 Public IP address
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
                echo 'Checking if index.html exists, or creating a fresh one...'
                sh """
                if [ ! -f index.html ]; then
                    echo '==== Creating index.html on-the-fly ===='
                    cat << 'EOF' > index.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Deployment Successful</title>
    <style>
        body { font-family: system-ui, sans-serif; background-color: #f4f7f6; display: flex; justify-content: center; align-items: center; height: 100vh; margin: 0; }
        .card { background: white; padding: 30px; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.05); text-align: center; }
        .icon { font-size: 48px; color: #2ecc71; margin-bottom: 15px; }
        h1 { color: #2c3e50; margin: 0 0 10px 0; }
        .badge { background-color: #e8f8f5; color: #1abc9c; padding: 6px 12px; border-radius: 20px; font-weight: bold; display: inline-block; }
    </style>
</head>
<body>
    <div class="card">
        <div class="icon">✓</div>
        <h1>Deployment Success!</h1>
        <p>Your HTML page was packaged via Jenkins and deployed to Amazon Linux Nginx.</p>
        <div class="badge">Region: ap-south-1</div>
    </div>
</body>
</html>
EOF
                fi
                """
                sh "zip -r package-${BUILD_NUMBER}.zip index.html"
            }
        }

        stage('Upload to S3') {
            steps {
                sh "aws s3 cp package-${BUILD_NUMBER}.zip ${env.S3_BUCKET}/package-${BUILD_NUMBER}.zip --region ${env.AWS_REGION}"
            }
        }

        stage('Configure VM & Deploy') {
            steps {
                echo "Deploying directly via Jenkins Secure Copy (SCP) pipelines..."
                
                withCredentials([sshUserPrivateKey(credentialsId: 'ec2-mumbai-key', keyFileVariable: 'TEMP_KEY')]) {
                    sh """
                    # 1. Push the package directly from Jenkins to the Target VM using SCP
                    scp -o StrictHostKeyChecking=no -i \$TEMP_KEY package-${BUILD_NUMBER}.zip ec2-user@${env.EC2_PUBLIC_IP}:/home/ec2-user/package.zip
                    
                    # 2. Run the internal system installations and extract commands remotely
                    ssh -o StrictHostKeyChecking=no -i \$TEMP_KEY ec2-user@${env.EC2_PUBLIC_IP} '
                        echo "==== Installing Dependencies ===="
                        sudo yum install git nginx unzip -y
                        
                        echo "==== Managing Services ===="
                        sudo systemctl start nginx
                        sudo systemctl enable nginx
                        
                        echo "==== Cleaning Old Web Files ===="
                        sudo rm -rf /usr/share/nginx/html/*
                        
                        echo "==== Deploying New Web Content ===="
                        sudo unzip /home/ec2-user/package.zip -d /usr/share/nginx/html/
                        
                        echo "==== Bypassing 403 Forbidden: Fixing Folder Ownership & Permissions ===="
                        sudo chown -R nginx:nginx /usr/share/nginx/html
                        sudo chmod -R 755 /usr/share/nginx/html
                        sudo chmod 755 /usr/share/nginx /usr/share /usr
                        sudo chcon -Rt httpd_sys_content_t /usr/share/nginx/html
                        
                        echo "==== Post-Deployment Clean up ===="
                        rm -f /home/ec2-user/package.zip
                        
                        echo "==== Reloading Nginx Configuration ===="
                        sudo systemctl reload nginx
                        echo "==== DEPLOYMENT COMPLETE ===="
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
