pipeline {
    agent any
    
    stages {
        stage('Build Backend Image') {
            steps {
                sh '''
                    echo "=== Building Docker Image ==="
                    docker build -t backend-app backend/
                    echo "=== Image built successfully ==="
                '''
            }
        }
        
        stage('Deploy Backend Containers') {
            steps {
                sh '''
                    echo "=== Starting backend containers ==="
                    
                    # Simple timestamp - use date command
                    TIMESTAMP=$(date +%s)
                    
                    # Use timestamp in container names
                    docker run -d --name backend1-$TIMESTAMP backend-app
                    docker run -d --name backend2-$TIMESTAMP backend-app
                    
                    echo "=== Running containers ==="
                    docker ps | grep backend
                    
                    # Save timestamp
                    echo $TIMESTAMP > timestamp.txt
                '''
            }
        }
        
        stage('Deploy NGINX') {
            steps {
                sh '''
                    echo "=== Setting up NGINX Load Balancer ==="
                    
                    # Read timestamp
                    TIMESTAMP=$(cat timestamp.txt)
                    
                    echo "Using backend containers: backend1-$TIMESTAMP and backend2-$TIMESTAMP"
                    
                    # Wait for backends
                    sleep 5
                    
                    # Create nginx config
                    cat > nginx.conf << EOF
upstream backend_servers {
    server backend1-$TIMESTAMP:80;
    server backend2-$TIMESTAMP:80;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://backend_servers;
    }
}
EOF
                    
                    # Remove old nginx
                    docker rm -f nginx-lb 2>/dev/null || true
                    
                    # Start nginx
                    docker run -d \
                        --name nginx-lb \
                        -p 80:80 \
                        -v $(pwd)/nginx.conf:/etc/nginx/conf.d/default.conf \
                        nginx:alpine
                    
                    echo "=== NGINX deployed ==="
                    docker ps | grep nginx
                '''
            }
        }
        
        stage('Test') {
            steps {
                sh '''
                    echo "=== Testing ==="
                    sleep 5
                    curl -s http://localhost || echo "Waiting for nginx..."
                    curl -s http://localhost/app.cpp || echo "App not ready"
                '''
            }
        }
    }
    
    post {
        success {
            echo "✅ Pipeline SUCCESS!"
        }
        failure {
            echo "❌ Pipeline FAILED"
            sh 'docker ps -a | tail -20'
        }
    }
}
