        stage('Configure VM & Deploy') {
            steps {
                echo "Connecting to Amazon Linux EC2 Server at ${env.EC2_PUBLIC_IP} using SSH Agent..."
                
                sshagent(['ec2-mumbai-key']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ec2-user@${env.EC2_PUBLIC_IP} '
                        echo "==== Updating System Packages via YUM ===="
                        sudo yum update -y
                        
                        echo "==== Installing Tools ===="
                        sudo yum install git nginx unzip -y
                        
                        echo "==== Starting Nginx Service ===="
                        sudo systemctl start nginx
                        sudo systemctl enable nginx
                        
                        echo "==== Cleaning Old Web Files ===="
                        sudo rm -rf /usr/share/nginx/html/*
                        
                        echo "==== Fetching Package from S3 ===="
                        aws s3 cp ${env.S3_BUCKET}/package-${BUILD_NUMBER}.zip ./package.zip --region ${env.AWS_REGION}
                        
                        echo "==== Deploying New Web Content ===="
                        sudo unzip package.zip -d /usr/share/nginx/html/
                        
                        echo "==== FIXING 403 FORBIDDEN: Applying Strict Permissions ===="
                        # 1. Grant ownership to the nginx user process
                        sudo chown -R nginx:nginx /usr/share/nginx/html
                        
                        # 2. Grant universal read and execute rights to the web files
                        sudo chmod -R 755 /usr/share/nginx/html
                        
                        # 3. Ensure the parent directories are executable so Nginx can traverse down to the files
                        sudo chmod 755 /usr/share/nginx /usr/share /usr
                        
                        # 4. Fix SELinux Context Security Blocks (Critical for Amazon Linux)
                        sudo chcon -Rt httpd_sys_content_t /usr/share/nginx/html
                        
                        echo "==== Post-Deployment Clean up ===="
                        rm package.zip
                        
                        echo "==== Reloading Nginx Configuration ===="
                        sudo systemctl reload nginx
                    '
                    """
                }
            }
        }
