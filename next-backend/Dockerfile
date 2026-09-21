# Use Node.js 20 LTS Alpine image
FROM node:20-alpine

WORKDIR /app

# Set environment
ENV NODE_ENV=production
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

# Install libc compatibility library (required for some Node packages)
RUN apk add --no-cache libc6-compat

# Install dependencies
COPY package.json package-lock.json* ./
RUN npm ci

# Copy source code
COPY . .

# Build Next.js application
RUN npm run build

# Expose internal port
EXPOSE 3000

# Start Next.js production server
CMD ["npm", "run", "start"]