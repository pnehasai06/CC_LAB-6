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
                    
                    # Run backend containers
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
                    
                    # Remove old nginx if exists
                    docker rm -f nginx-lb 2>/dev/null || true
                    
                    # Create nginx config file in workspace
                    cat > nginx-lb.conf << EOF
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
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }
}
EOF
                    
                    echo "=== NGINX Configuration ==="
                    cat nginx-lb.conf
                    
                    # Start nginx with config mounted as volume
                    docker run -d \
                        --name nginx-lb \
                        -p 8081:80 \
                        -v $(pwd)/nginx-lb.conf:/etc/nginx/conf.d/default.conf \
                        --link backend1-$TIMESTAMP \
                        --link backend2-$TIMESTAMP \
                        nginx:alpine
                    
                    echo "=== NGINX deployed successfully on port 8081 ==="
                    docker ps | grep nginx
                '''
            }
        }
        
        stage('Test') {
            steps {
                sh '''
                    echo "=== Testing Load Balancer on port 8081 ==="
                    sleep 5
                    
                    echo "Testing multiple requests to see load balancing:"
                    for i in 1 2 3 4 5; do
                        echo "Request $i:"
                        curl -s http://localhost:8081 | head -2 || echo "Waiting..."
                        sleep 1
                    done
                    
                    echo "=== Testing app.cpp endpoint ==="
                    curl -s http://localhost:8081/app.cpp || echo "App endpoint not ready"
                '''
            }
        }
    }
    
    post {
        always {
            echo "=== Pipeline Complete ==="
        }
        success {
            echo "✅ SUCCESS! Load balancer is running at http://localhost:8081"
            sh '''
                echo "Running containers:"
                docker ps --format "table {{.Names}}\t{{.Status}}" | grep -E "backend|nginx" || echo "No containers found"
            '''
        }
        failure {
            echo "❌ FAILED - Debug Information:"
            sh '''
                echo "=== Current Containers ==="
                docker ps -a | tail -10
                
                echo "=== Port 8081 Status ==="
                sudo lsof -i :8081 || echo "Port 8081 is free"
                
                echo "=== Last 20 lines of container logs ==="
                docker logs nginx-lb --tail 20 2>/dev/null || echo "No nginx-lb container"
                
                echo "=== Timestamp value ==="
                cat timestamp.txt 2>/dev/null || echo "No timestamp file"
                
                echo "=== nginx config file content ==="
                cat nginx-lb.conf 2>/dev/null || echo "No nginx-lb.conf file"
            '''
        }
    }
}
