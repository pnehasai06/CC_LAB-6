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
                    
                    # Simple timestamp
                    TIMESTAMP=$(date +%s)
                    
                    # Create a network for containers to communicate
                    docker network create backend-net-$TIMESTAMP 2>/dev/null || true
                    
                    # Run backend containers on the network
                    docker run -d \
                        --name backend1-$TIMESTAMP \
                        --network backend-net-$TIMESTAMP \
                        backend-app
                    
                    docker run -d \
                        --name backend2-$TIMESTAMP \
                        --network backend-net-$TIMESTAMP \
                        backend-app
                    
                    echo "=== Running containers ==="
                    docker ps | grep backend
                    
                    # Save timestamp
                    echo $TIMESTAMP > timestamp.txt
                    echo backend-net-$TIMESTAMP > network.txt
                '''
            }
        }
        
        stage('Deploy NGINX') {
            steps {
                sh '''
                    echo "=== Setting up NGINX Load Balancer ==="
                    
                    # Read timestamp
                    TIMESTAMP=$(cat timestamp.txt)
                    NETWORK=$(cat network.txt)
                    
                    echo "Using network: $NETWORK"
                    echo "Using backend containers: backend1-$TIMESTAMP and backend2-$TIMESTAMP"
                    
                    # Wait for backends
                    sleep 5
                    
                    # Remove old nginx if exists
                    docker rm -f nginx-lb 2>/dev/null || true
                    
                    # Start nginx on the same network
                    docker run -d \
                        --name nginx-lb \
                        --network $NETWORK \
                        -p 80:80 \
                        nginx:alpine
                    
                    # Create nginx config inside the container
                    docker exec nginx-lb sh -c "cat > /etc/nginx/conf.d/default.conf << 'EOF'
upstream backend_servers {
    server backend1-$TIMESTAMP:80;
    server backend2-$TIMESTAMP:80;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://backend_servers;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
    }
}
EOF"
                    
                    # Reload nginx
                    docker exec nginx-lb nginx -s reload
                    
                    echo "=== NGINX deployed successfully ==="
                    docker ps | grep nginx
                '''
            }
        }
        
        stage('Test') {
            steps {
                sh '''
                    echo "=== Testing Load Balancer ==="
                    sleep 5
                    
                    echo "Testing multiple requests to see load balancing:"
                    for i in 1 2 3 4 5; do
                        echo "Request $i:"
                        curl -s http://localhost | grep -o "backend[0-9]-[0-9]*" || echo "Response: $(curl -s http://localhost | head -1)"
                        sleep 1
                    done
                    
                    echo "=== Testing app.cpp endpoint ==="
                    curl -s http://localhost/app.cpp | head -5
                '''
            }
        }
    }
    
    post {
        always {
            echo "=== Pipeline Complete ==="
        }
        success {
            echo "✅ SUCCESS! Load balancer is running at http://localhost"
        }
        failure {
            echo "❌ FAILED - Check logs above"
            sh '''
                echo "=== Debug Info ==="
                docker ps -a | tail -10
                docker network ls | tail -5
            '''
        }
    }
}
