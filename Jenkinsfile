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
            
            # Start nginx with links to backend containers
            docker run -d \
                --name nginx-lb \
                -p 80:80 \
                --link backend1-$TIMESTAMP \
                --link backend2-$TIMESTAMP \
                nginx:alpine
            
            # Create nginx config inside the container - FIXED VERSION
            docker exec nginx-lb sh -c "cat > /etc/nginx/conf.d/default.conf << 'EOF'
upstream backend_servers {
    server backend1-$TIMESTAMP:80;
    server backend2-$TIMESTAMP:80;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://backend_servers;
        proxy_set_header Host \\$host;
        proxy_set_header X-Real-IP \\$remote_addr;
    }
}
EOF"
            
            # Test nginx configuration
            docker exec nginx-lb nginx -t
            
            # Reload nginx if config is valid
            docker exec nginx-lb nginx -s reload || true
            
            echo "=== NGINX deployed successfully ==="
            docker ps | grep nginx
        '''
    }
}
