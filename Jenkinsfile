pipeline {
    agent any
    
    environment {
        // Create a unique timestamp for this build
        TIMESTAMP = "${System.currentTimeMillis() / 1000}".toInteger().toString()
    }
    
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
                    echo "=== Starting backend containers with unique names ==="
                    echo "Using timestamp: ${TIMESTAMP}"
                    
                    # Use timestamp in container names to avoid conflicts
                    docker run -d --name backend1-${TIMESTAMP} backend-app
                    docker run -d --name backend2-${TIMESTAMP} backend-app
                    
                    echo "=== Running containers ==="
                    docker ps | grep backend
                    
                    # Save timestamp for later stages
                    echo ${TIMESTAMP} > timestamp.txt
                '''
            }
        }
        
        stage('Deploy NGINX Load Balancer') {
            steps {
                sh '''
                    echo "=== Setting up NGINX Load Balancer ==="
                    
                    # Read timestamp
                    if [ -f timestamp.txt ]; then
                        TIMESTAMP=$(cat timestamp.txt)
                    else
                        TIMESTAMP=${TIMESTAMP}
                    fi
                    
                    echo "Using backend containers: backend1-${TIMESTAMP} and backend2-${TIMESTAMP}"
                    
                    # Wait for backend containers to be ready
                    sleep 5
                    
                    # Create nginx config with dynamic backend names
                    cat > nginx-lb-${TIMESTAMP}.conf << EOF
upstream backend_servers {
    server backend1-${TIMESTAMP}:80;
    server backend2-${TIMESTAMP}:80;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://backend_servers;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
    }
}
EOF
                    
                    echo "=== NGINX Configuration ==="
                    cat nginx-lb-${TIMESTAMP}.conf
                    
                    # Remove old nginx container (ignore errors)
                    docker rm -f nginx-lb 2>/dev/null || true
                    
                    # Start nginx container with config
                    docker run -d \
                        --name nginx-lb \
                        -p 80:80 \
                        -v $(pwd)/nginx-lb-${TIMESTAMP}.conf:/etc/nginx/conf.d/default.conf \
                        nginx:alpine
                    
                    echo "=== NGINX Load Balancer deployed ==="
                    
                    # Show running containers
                    docker ps | grep -E "backend|nginx"
                '''
            }
        }
        
        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "=== Verifying Deployment ==="
                    
                    # Read timestamp
                    if [ -f timestamp.txt ]; then
                        TIMESTAMP=$(cat timestamp.txt)
                    fi
                    
                    echo "Running containers:"
                    docker ps | grep -E "backend|nginx"
                    
                    echo "=== Testing NGINX Load Balancer ==="
                    sleep 5
                    
                    # Test multiple times to see load balancing
                    echo "Request 1:"
                    curl -s http://localhost | grep -o "backend[0-9]-[0-9]*" || echo "Response doesn't show backend"
                    
                    echo "Request 2:"
                    curl -s http://localhost | grep -o "backend[0-9]-[0-9]*" || echo "Response doesn't show backend"
                    
                    echo "Request 3:"
                    curl -s http://localhost | grep -o "backend[0-9]-[0-9]*" || echo "Response doesn't show backend"
                    
                    echo "=== Testing app.cpp endpoint ==="
                    curl -s http://localhost/app.cpp | head -5
                '''
            }
        }
    }
    
    post {
        always {
            echo "=== Pipeline Completed ==="
        }
        success {
            echo "✅ Pipeline executed successfully for PES1UG23AAM200!"
            sh '''
                echo "=== Final Running Containers ==="
                docker ps | grep -E "backend|nginx"
                echo ""
                echo "=== Test Commands ==="
                echo "curl http://localhost"
                echo "curl http://localhost/app.cpp"
                echo ""
                echo "Open in browser: http://localhost"
            '''
        }
        failure {
            echo "❌ Pipeline failed. Check logs above for errors."
            sh '''
                echo "=== Debug Information ==="
                echo "Docker containers:"
                docker ps -a | tail -20
            '''
        }
    }
}
