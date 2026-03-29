# 1️⃣ Use a lightweight Node.js image
FROM node:20-alpine

# 2️⃣ Set working directory
WORKDIR /app

# 3️⃣ Copy package files
COPY package*.json ./

# 4️⃣ Install dependencies (works even if package-lock.json is missing)
RUN npm install --only=production

# 5️⃣ Copy application files
COPY . .

# 6️⃣ Expose port (if your app listens on 3000, change if needed)
EXPOSE 3000

# 7️⃣ Default command
CMD ["node", "app.js"]
