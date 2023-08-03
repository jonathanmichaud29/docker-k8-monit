#FROM nginx:stable-alpine
#COPY my-app/build/ /usr/share/nginx/html

FROM node:18.12.1-alpine

# set working directory
WORKDIR /app

# add `/app/node_modules/.bin` to $PATH
ENV PATH /app/node_modules/.bin:$PATH

# install app dependencies
COPY my-app/package.json ./
COPY my-app/package-lock.json ./
RUN npm install

# add app
COPY . ./

# start app
CMD ["node", "server.js"]
