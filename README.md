Dockerizing a Vue App
Last update: May 21, 2019 • docker, vue

This tutorial looks at how to Dockerize a Vue app, built with the Vue CLI, using Docker along with Docker Compose and Docker Machine for both development and production. We’ll specifically focus on-

Setting up a development environment with code hot-reloading
Configuring a production-ready image using multistage builds
We will be using:

Docker v18.09.2
Vue CLI v3.7.0
Node v12.2.0
Contents
Project Setup
Docker
Docker Machine
Production
Vue Router and Nginx
Project Setup
Install the Vue CLI globally:

$ npm install -g @vue/cli@3.7.0
Generate a new app, using the default preset:

$ vue create my-app --default
$ cd my-app
Docker
Add a Dockerfile to the project root:

# base image
FROM node:12.2.0-alpine

# set working directory
WORKDIR /app

# add `/app/node_modules/.bin` to $PATH
ENV PATH /app/node_modules/.bin:$PATH

# install and cache app dependencies
COPY package.json /app/package.json
RUN npm install
RUN npm install @vue/cli@3.7.0 -g

# start app
CMD ["npm", "run", "serve"]
Add a .dockerignore as well:

node_modules
.git
.gitignore
This will speed up the Docker build process as our local dependencies and git repo will not be sent to the Docker daemon.

Build and tag the Docker image:

$ docker build -t my-app:dev .
Then, spin up the container once the build is done:

$ docker run -v ${PWD}:/app -v /app/node_modules -p 8081:8080 --rm my-app:dev
What’s happening here?

The docker run command creates a new container instance, from the image we just created, and runs it.
-v ${PWD}:/app mounts the code into the container at “/app”.

{PWD} may not work on Windows. See this Stack Overflow question for more info.

Since we want to use the container version of the “node_modules” folder, we configured another volume: -v /app/node_modules. You should now be able to remove the local “node_modules” flavor.
-p 8081:8080 exposes port 8080 to other Docker containers on the same network (for inter-container communication) and port 8081 to the host.

For more, review this Stack Overflow question.

Finally, --rm removes the container and volumes after the container exits.
Open your browser to http://localhost:8081 and you should see the app. Try making a change to the App component (src/App.vue) within your code editor. You should see the app hot-reload. Kill the server once done.

What happens when you add -it?

$ docker run -it -v ${PWD}:/app -v /app/node_modules -p 8081:8080 --rm my-app:dev
Check your understanding and look this up on your own.

Want to use Docker Compose? Add a docker-compose.yml file to the project root:

version: '3.7'

services:

  my-app:
    container_name: my-app
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - '.:/app'
      - '/app/node_modules'
    ports:
      - '8081:8080'
Take note of the volumes. Without the anonymous volume ('/app/node_modules'), the node_modules directory would be overwritten by the mounting of the host directory at runtime. In other words, this would happen:

Build - The node_modules directory is created in the image.
Run - The current directory is mounted into the container, overwriting the node_modules that were installed during the build.
Build the image and fire up the container:

$ docker-compose up -d --build
Ensure the app is running in the browser and test hot-reloading again. Bring down the container before moving on:

$ docker-compose stop
Windows Users: Having problems getting the volumes to work properly? Review the following resources:

Docker on Windows–Mounting Host Directories
Configuring Docker for Windows Shared Drives
You also may need to add COMPOSE_CONVERT_WINDOWS_PATHS=1 to the environment portion of your Docker Compose file. Review the Declare default environment variables in file guide for more info.

Docker Machine
To get hot-reloading to work with Docker Machine and VirtualBox you’ll need to enable a polling mechanism via chokidar (which wraps fs.watch, fs.watchFile, and fsevents).

Create a new Machine:

$ docker-machine create -d virtualbox my-app
$ docker-machine env my-app
$ eval $(docker-machine env my-app)
Grab the IP address:

$ docker-machine ip my-app
Then, build the images:

$ docker build -t my-app:dev .
And run the container:

$ docker run -it -v ${PWD}:/app -v /app/node_modules -p 8081:8080 --rm my-app:dev
Test the app again in the browser at http://DOCKER_MACHINE_IP:8081 (make sure to replace DOCKER_MACHINE_IP with the actual IP address of the Docker Machine). Also, confirm that auto reload is not working. You can try with Docker Compose as well, but the result will be the same.

To get hot-reload working, we need to add an environment variable: CHOKIDAR_USEPOLLING=true.

$ docker run -it -v ${PWD}:/app -v /app/node_modules -p 8081:8080 -e CHOKIDAR_USEPOLLING=true --rm my-app:dev
Test it out again. Then, kill the server and add the environment variable to the docker-compose.yml file:

version: '3.7'

services:

  my-app:
    container_name: my-app
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - '.:/app'
      - '/app/node_modules'
    ports:
      - '8081:8080'
    environment:
      - CHOKIDAR_USEPOLLING=true
Production
Let’s create a separate Dockerfile for use in production called Dockerfile-prod:

# build environment
FROM node:12.2.0-alpine as build
WORKDIR /app
ENV PATH /app/node_modules/.bin:$PATH
COPY package.json /app/package.json
RUN npm install --silent
RUN npm install @vue/cli@3.7.0 -g
COPY . /app
RUN npm run build

# production environment
FROM nginx:1.16.0-alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
Here, we take advantage of the multistage build pattern to create a temporary image used for building the artifact – the production-ready Vue static files – that is then copied over to the production image. The temporary build image is discarded along with the original files and folders associated with the image. This produces a lean, production-ready image.

Check out the Builder pattern vs. Multi-stage builds in Docker blog post for more info on multistage builds.

Using the production Dockerfile, build and tag the Docker image:

$ docker build -f Dockerfile-prod -t my-app:prod .
Spin up the container:

$ docker run -it -p 80:80 --rm my-app:prod
Assuming you are still using the same Docker Machine, navigate to http://DOCKER_MACHINE_IP/ in your browser.

Test with a new Docker Compose file as well called docker-compose-prod.yml:

version: '3.7'

services:

  my-app-prod:
    container_name: my-app-prod
    build:
      context: .
      dockerfile: Dockerfile-prod
    ports:
      - '80:80'
Fire up the container:

$ docker-compose -f docker-compose-prod.yml up -d --build
Test it out once more in your browser.

If you’re done, go ahead and destroy the Machine:

$ eval $(docker-machine env -u)
$ docker-machine rm my-app
Vue Router and Nginx
If you’re using Vue Router, then you’ll need to change the default Nginx config at build time:

RUN rm /etc/nginx/conf.d/default.conf
COPY nginx/nginx.conf /etc/nginx/conf.d
Add the changes to Dockerfile-prod:

# build environment
FROM node:12.2.0-alpine as build
WORKDIR /app
ENV PATH /app/node_modules/.bin:$PATH
COPY package.json /app/package.json
RUN npm install --silent
RUN npm install @vue/cli@3.7.0 -g
COPY . /app
RUN npm run build

# production environment
FROM nginx:1.16.0-alpine
COPY --from=build /app/dist /usr/share/nginx/html
RUN rm /etc/nginx/conf.d/default.conf
COPY nginx/nginx.conf /etc/nginx/conf.d
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
Create the following folder along with a nginx.conf file:

└── nginx
    └── nginx.conf
nginx.conf:

server {

  listen 80;

  location / {
    root   /usr/share/nginx/html;
    index  index.html index.htm;
    try_files $uri $uri/ /index.html;
  }

  error_page   500 502 503 504  /50x.html;

  location = /50x.html {
    root   /usr/share/nginx/html;
  }

}
Cheers!
