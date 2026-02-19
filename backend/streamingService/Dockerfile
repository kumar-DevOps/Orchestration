FROM node:18-alpine AS base

WORKDIR /app/streamingService

# Copy package.json and package-lock.json first
COPY package*.json ./

# Install dependencies
RUN npm install --production

# Copy the rest of the service code
COPY . .

ENV NODE_ENV=production
EXPOSE 3001

CMD ["npm", "run", "start"]

