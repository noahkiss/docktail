FROM nginx:alpine

# Copy custom nginx config that accepts any host
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy custom HTML
COPY index.html /usr/share/nginx/html/index.html

# Create a script to generate hostname file on startup
RUN echo '#!/bin/sh' > /docker-entrypoint.d/99-hostname.sh && \
    echo 'hostname > /usr/share/nginx/html/hostname.txt' >> /docker-entrypoint.d/99-hostname.sh && \
    chmod +x /docker-entrypoint.d/99-hostname.sh
