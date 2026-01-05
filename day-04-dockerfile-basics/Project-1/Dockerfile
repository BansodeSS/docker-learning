# Use official nginx image as base
FROM nginx:alpine

# Copy our HTML file to nginx html directory
COPY index.html /usr/share/nginx/html/index.html

# Expose port 80
EXPOSE 80

# Start nginx (CMD is inherited from base image)