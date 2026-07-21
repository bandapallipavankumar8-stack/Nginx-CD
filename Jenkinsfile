pipeline {
    agent any

    environment {
        S3_BUCKET = 's3://nginx-ci/packages'
        AWS_REGION = 'ap-south-1'
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
        <p>Your HTML page was packaged via Jenkins and deployed to Nginx.</p>
        <div class="badge">Region: ap-south-1</div>
    </div>
</body>
</html>
EOF
                fi
                """
                
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
