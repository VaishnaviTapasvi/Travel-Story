FROM nexus-service-for-docker-hosted-registry.nexus.svc.cluster.local:8085/library/node:18


WORKDIR /app

COPY package*.json ./
RUN npm install --production
COPY . .
RUN npm rebuild bcrypt --build-from-source


EXPOSE 5000
CMD ["node", "index.js"]
