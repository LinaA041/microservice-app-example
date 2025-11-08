# Etapa de construcción
FROM node:8.17.0 AS builder

WORKDIR /app

# Copiar dependencias y luego el código
COPY package*.json ./
RUN npm install

COPY . .

# Exponer el puerto configurado por la app
EXPOSE 8082

# Variables de entorno
ENV JWT_SECRET=PRFT \
    TODO_API_PORT=8082 \
    REDIS_HOST=redis-service \
    REDIS_PORT=6379 \
    REDIS_CHANNEL=log_channel

# Comando de ejecución
CMD ["npm", "start"]
