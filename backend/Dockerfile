# Stage 1: Build stage
FROM ubuntu:20.04 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .

# Stage 2: Run stage (lightweight image)
FROM ubuntu:20.04
WORKDIR /app
COPY --from=builder /app .
EXPOSE 5000
CMD ["node", "server.js"]
