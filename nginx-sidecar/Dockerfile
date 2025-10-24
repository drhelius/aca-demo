FROM nginx:alpine

# Copy nginx configuration
COPY nginx.conf /etc/nginx/nginx.conf

# Copy htpasswd file for basic auth
COPY .htpasswd /etc/nginx/.htpasswd

# Expose port
EXPOSE 8080

CMD ["nginx", "-g", "daemon off;"]
