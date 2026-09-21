# Stage 1: Build Vite React project
FROM node:20-alpine AS builder

WORKDIR /app

# Install dependencies
COPY package.json package-lock.json* ./
RUN npm ci

# Copy source files
COPY . .

# Set empty VITE_API_URL so that API calls use relative paths (e.g., fetch('/api/...'))
ENV VITE_API_URL=""

# Build production assets into dist/
RUN npm run build

# Stage 2: Serve static bundle using Nginx
FROM nginx:alpine

# Copy custom Nginx configuration
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy compiled static files from the builder stage
COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]