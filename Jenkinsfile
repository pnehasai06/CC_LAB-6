pipeline {
    agent any
    
    environment {
        // Set a common timestamp for all stages
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
                    echo "=== Removing old containers (if possible) ==="
                    docker rm -f backend1 backend2 2>/dev/null || true
                    
                    echo "=== Starting new containers with unique names ==="
                    echo "Using timestamp: ${TIMESTAMP}"
                    
                    docker run -d --name backend1-${TIMESTAMP} backend-app
                    docker run -d --name backend2-${TIMESTAMP} backend-app
                    
                    echo "=== Running containers ==="
                    docker ps | grep backend
                    
                    # Save timestamp for next stages
                    echo ${TIMESTAMP} > timestamp.txt
                '''
            }
        }
        
        stage('Deploy NGINX Load Balancer') {
            steps {
                sh '''
                    echo "=== Setting up NGINX Load Balancer ==="
                    
                    # Read the timestamp from file
                    if [ -f timestamp.txt ]; then
                        TIMESTAMP=$(cat timestamp.txt)
                    else
                        TIMESTAMP=${TIMESTAMP}
                    fi
                    
                    BACKEND1_NAME="backend1-${TIMESTAMP}"
                    BACKEND2_NAME="backend2-${TIMESTAMP}"
                    
                    echo "Using backend containers: ${BACKEND1_NAME} and ${BACKEND2_NAME}"
                    
                    # Remove old nginx container if exists
                    docker rm -f nginx-lb 2>/dev/null || true
                    
                    # Create nginx config dynamically
                    cat > nginx-lb.conf << EOF
upstream backend_servers {
    server ${BACKEND1_NAME}:80;
    server ${BACKEND2_NAME}:80;
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
                    
                    # Show the config
                    echo "=== NGINX Configuration ==="
                    cat nginx-lb.conf
                    
                    # Start nginx container with the config
                    docker run -d \
                        --name nginx-lb \
                        -p 80:80 \
                        -v $(pwd)/nginx-lb.conf:/etc/nginx/conf.d/default.conf \
                        --link ${BACKEND1_NAME} \
                        --link ${BACKEND2_NAME} \
                        nginx:alpine
                    
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
                    curl -I http://localhost 2>/dev/null || echo "NGINX is running but may need a moment to start"
                    
                    echo "=== Testing Backend Access ==="
                    curl http://localhost/app.cpp 2>/dev/null || echo "App endpoint not yet available"
                '''
            }
        }
    }
    
    post {
        always {
            echo "=== Pipeline Execution Summary ==="
            echo "Pipeline completed. Check logs for details."
        }
        success {
            echo "✅ Pipeline executed successfully!"
            sh '''
                echo "=== Final Running Containers ==="
                docker ps | grep -E "backend|nginx"
            '''
        }
        failure {
            echo "❌ Pipeline failed. Check console logs for errors."
            sh '''
                echo "=== Debug Information ==="
                echo "Current directory: $(pwd)"
                echo "Docker images:"
                docker images | grep backend
                echo "Docker containers (all):"
                docker ps -a | grep -E "backend|nginx"
                echo "=== Timestamp file contents ==="
                if [ -f timestamp.txt ]; then
                    cat timestamp.txt
                else
                    echo "No timestamp file found"
                fi
            '''
        }
        cleanup {
            echo "=== Cleanup Steps ==="
            // Uncomment to auto-cleanup
            // sh '''
            //     echo "Cleaning up containers..."
            //     docker rm -f nginx-lb 2>/dev/null || true
            //     docker rm -f $(docker ps -a | grep backend | awk '{print $1}') 2>/dev/null || true
            // '''
        }
    }
}
