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
                    echo "=== Cleaning up old containers ==="
                    # Force remove any existing containers
                    docker rm -f backend1 backend2 nginx-lb 2>/dev/null || true
                    
                    echo "=== Starting backend containers ==="
                    # Use simple names without timestamps
                    docker run -d --name backend1 backend-app
                    docker run -d --name backend2 backend-app
                    
                    echo "=== Running containers ==="
                    docker ps | grep -E "backend|nginx"
                '''
            }
        }
        
        stage('Deploy NGINX Load Balancer') {
            steps {
                sh '''
                    echo "=== Setting up NGINX Load Balancer ==="
                    
                    # Wait for backend containers to be ready
                    sleep 5
                    
                    # Create nginx config
                    cat > nginx-lb.conf << 'EOF'
upstream backend_servers {
    server backend1:80;
    server backend2:80;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://backend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF
                    
                    echo "=== NGINX Configuration ==="
                    cat nginx-lb.conf
                    
                    # Start nginx container
                    docker run -d \
                        --name nginx-lb \
                        -p 80:80 \
                        -v $(pwd)/nginx-lb.conf:/etc/nginx/conf.d/default.conf \
                        nginx:alpine
                    
                    # Connect nginx to backend containers network
                    docker network connect bridge nginx-lb 2>/dev/null || true
                    
                    echo "=== NGINX Load Balancer deployed ==="
                '''
            }
        }
        
        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "=== Verifying Deployment ==="
                    echo "Running containers:"
                    docker ps | grep -E "backend|nginx"
                    
                    echo "=== Testing NGINX Load Balancer ==="
                    sleep 5
                    curl -s -I http://localhost | head -n 1 || echo "Waiting for NGINX to start..."
                    
                    echo "=== Testing Backend Access ==="
                    sleep 2
                    curl -s http://localhost/app.cpp || echo "App endpoint not yet available"
                '''
            }
        }
    }
    
    post {
        always {
            echo "=== Pipeline Completed ==="
        }
        success {
            echo "✅ Pipeline executed successfully!"
            sh '''
                echo "=== Final Running Containers ==="
                docker ps | grep -E "backend|nginx"
            '''
        }
        failure {
            echo "❌ Pipeline failed. Check logs above for errors."
            sh '''
                echo "=== Debug Information ==="
                echo "Current directory: $(pwd)"
                echo "Docker images:"
                docker images | grep backend
                echo "Docker containers (all):"
                docker ps -a | grep -E "backend|nginx"
            '''
        }
    }
}
