FROM node:latest as build-stage
WORKDIR /app
COPY ./package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:1.17 as production-stage
RUN echo "hello1"
COPY nginx.conf /etc/nginx/nginx.conf
RUN echo "hello2"
RUN mkdir /app
COPY --from=build-stage /app/dist /app
EXPOSE 8080:8080
CMD ["nginx", "-g", "daemon off;"]
