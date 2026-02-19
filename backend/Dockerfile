# Use a more complete base image
FROM alpine:latest

# Install nginx and g++
RUN apk add --no-cache nginx g++

# Copy application
COPY app.cpp /usr/share/nginx/html/app.cpp

# Compile the app
RUN g++ /usr/share/nginx/html/app.cpp -o /usr/share/nginx/html/app

# Copy nginx config
COPY nginx.conf /etc/nginx/http.d/default.conf

# Expose port
EXPOSE 80

# Start nginx
CMD ["nginx", "-g", "daemon off;"]

