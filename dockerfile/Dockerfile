# Base image
FROM ubuntu:20.04

# Maintainer label
LABEL maintainer="Ramtin"

# Set non-interactive mode for apt
ENV DEBIAN_FRONTEND=noninteractive

# Update & install nginx, clean up to reduce image size
RUN apt update && \
    apt install -y nginx curl tzdata && \
    rm -rf /var/lib/apt/lists/*

# Set timezone (optional but recommended)
ENV TZ=Asia/Tehran

# Add default welcome page
RUN echo "<h1>Welcome to Nginx on Ubuntu Docker</h1>" > /var/www/html/index.html

# Ensure nginx runs in the foreground
CMD ["nginx", "-g", "daemon off;"]

# Expose HTTP port
EXPOSE 80
