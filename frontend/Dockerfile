FROM node:8.17.0

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

# Variables de entorno para configuración del frontend
ENV PORT=8080 \
    AUTH_API_ADDRESS=http://localhost:8000 \
    TODOS_API_ADDRESS=http://localhost:8082

EXPOSE 8080

CMD ["npm", "start"]
