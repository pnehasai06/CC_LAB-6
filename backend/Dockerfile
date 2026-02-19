FROM nginx:alpine

# Copy nginx configuration
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy the C++ application
COPY app.cpp /usr/share/nginx/html/app.cpp

# Expose port 80
EXPOSE 80
