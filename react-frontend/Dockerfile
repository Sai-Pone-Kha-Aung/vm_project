# Build stage 

FROM node:22-alpine AS build 

WORKDIR /app 
 
COPY package*.json ./ 

RUN npm ci 
 
COPY . . 

# Vite inlines VITE_* vars at build time; pass via --build-arg to override .env files 

ARG VITE_API_URL 

ENV VITE_API_URL=$VITE_API_URL 

RUN npm run build 

 
# Production stage 

FROM nginx:alpine AS production 

COPY --from=build /app/dist /usr/share/nginx/html 

COPY nginx.conf /etc/nginx/conf.d/default.conf 

 

EXPOSE 80 

CMD ["nginx", "-g", "daemon off;"] 