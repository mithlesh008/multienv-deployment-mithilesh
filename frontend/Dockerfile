FROM node:20-alpine AS build

WORKDIR /app

COPY package*.json ./

RUN npm install --legacy-peer-deps

# Match react-scripts 5 / schema-utils 3
RUN npm install --no-save --legacy-peer-deps \
    ajv@6.12.6 \
    ajv-keywords@3.5.2

COPY . .

ENV CI=false
ENV NODE_OPTIONS=--openssl-legacy-provider

RUN npm run build

FROM node:20-alpine

RUN npm install -g serve

WORKDIR /app

COPY --from=build /app/build ./build

EXPOSE 3000

CMD ["serve", "-s", "build", "-l", "3000"]

