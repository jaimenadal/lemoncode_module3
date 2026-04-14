# Etapa 1: Builder
FROM node:22-alpine AS builder

WORKDIR /app

# Copiar manifiestos de dependencias primero
COPY todo-app/package*.json ./
RUN npm ci --omit=dev

COPY todo-app/frontend/package*.json ./frontend/
RUN cd frontend && npm ci

# Copiar el resto del código fuente
COPY todo-app/ ./

# Build del frontend. webpack genera el output en frontend/dist
RUN cd frontend && npm run build

# Etapa 2: Runtime
FROM node:22-alpine AS runtime

WORKDIR /app

# Solo lo necesario para producción
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/src ./src

# El frontend compilado va a /app/public. ruta que sirve express.static('public')
COPY --from=builder /app/frontend/dist ./public

# Variables de entorno con valores por defecto
ENV NODE_ENV=production
ENV PORT=3000

EXPOSE 3000

CMD ["node", "src/index.js"]
