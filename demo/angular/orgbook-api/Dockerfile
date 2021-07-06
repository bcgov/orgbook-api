# Stage: build
FROM node:10 as build
WORKDIR /app
COPY package*.json /app/
RUN npm install
COPY . .
ARG RUNMODE
ARG ORGBOOK_HOST=${ORGBOOK_HOST:-"orgbook.gov.bc.ca"}
RUN npm run build

# Stage: runtime
FROM nginx:1.21-alpine as runtime
RUN rm -rf /usr/share/nginx/html/*
COPY nginx.conf /etc/nginx/nginx.conf
COPY --from=build /app/dist/orgbook-api /usr/share/nginx/html/
ENTRYPOINT ["nginx", "-g", "daemon off;"]