# Build Stage
FROM nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085/library/node:18 AS build


WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Production Stage
FROM nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085/library/nginx:alpine

COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
