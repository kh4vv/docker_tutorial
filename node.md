
# Build your Node Image
## Overview
To complete the tutorial, you need following:
* Node.js version 12.18 or later. [Download Node.js](https://nodejs.org/en/)
* Docker running locally: Follow the instructions to [download and install Docker](https://docs.docker.com/desktop/)
* An IDE or a text editor to edit files. We recommend using Visual Studio Code.

## Sample application
Let's create a simple Node.js applications that we use as our example. Create a directory in your local machine named `node-docker` and follow the steps below to create  a simple REST API.

```
$ cd [path to your node-directory]
$ npm init -y
$ npm install ronin-server ronin-mocks
$ touch server.js
```

Open this working directory in your IDE and add the following code into the `server.js`file.
```
const ronin = require('ronin-server')
const mocks = require('ronin-mocks')

const server = ronin.server()

server.use('/', mocks.server(server.Router(), false, true) )
server.start()
```
The mocking server is called `Ronin.js` and will listen on port 8000 by default

## Test the application
Let's start our application and make sure it's running properly.
```
$ node server.js
```
To test that the application is working properly, we'll first POST some JSON to the API and then make a GET request to see that the data has been saved. 

```
$ curl --request POST --url http://localhost:8000/test --header 'content-type: application/json' --data '{"msg: testing"}' {"code":"success","payload":[{"msg":"testing","id":```
"31f23305-f5d0-4b4f-a16f-6f4c8ec93cf1","createDate":"2020-08-28T21:53:07.157Z"}]}
$curl https://localhost:8000/test ```
{"code":"success","meta":{"total":1,"count":1},"payload":[{"msg":"testing","id":"31f23305-f5d0-4b4f-a16f-6f4c8ec93cf1","createDate":"2020-08-28T21:53:07.157Z"}]}
```
Switching back to the terminal where our server is running. You should now see the following request in the server logs.
```
2020-XX-31T16:35:08:4260  INFO: POST /test
2020-XX-31T16:35:21:3560  INFO: GET /test
```

## Create a Dockerfile for Node.js
A Dockerfile is a text document that contains all the commands a user could call on the command line to assemble an Image. 
The first thing is to add a line our Dockerfile that tells Docker what base image we would like to use for the appliation.
```
FROM node:12.18.1
```
Docker image can be inherited from other images. When we use the `FROM` command, we tell Docker to include in our image all the functionality from the `node:12.18.1` image. 
The `NODE_ENV` environment variable specifies the environment in which an application is running. 
```
ENV NODE_ENV=production
```
Then we should set up the working directory. 
```
WORKDIR /app
```
Before we run `npm install`, we need to get our `package.json` and `package-lock.json` files into our image. We use the `COPY` command to do this. The `COPY`command takes two parameters. The first parameter tells Docker what file(s) you would like to copy into the image. The second parameter tells Docker where you want that file(s) to be copied to. 
```
COPY ["package.json", "package-lock.json*", "./"]
```
Then we can use `RUN` command to execute the command npm install. 
```
RUN npm install --production
```
At this point, we have a image that is based on node version 12.18.1 and we have installed our dependencies. The next thing is to add our source code into the image.
```
COPY . .
```
 Now, we tell Docker what command we want to run when our image is run inside of the container. 
 ```
 CMD ["node", "server.js"]
 ```
 Here is the completed Dockerfile
 ```
FROM node:12.18.1
ENV NODE_ENV=production

WORKDIR /app

COPY ["package.json", "package-lock.json*", "./"]

RUN npm install --production

COPY . .

CMD [ "node", "server.js" ]
```

## Build Image
The `docker build` command builds Docker images from a Dockerfile and a "context". 
The build command optionally takes a `--tag` flag which is used to set the name of the image. 
```
docker build --tag node-docker .
```
## View Local Images
To list images, simply run `images` command
```
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
node-docker         latest              3809733582bc        About a minute ago   945MB
node                12.18.1             f5be1883c8e0        2 months ago         918MB
```
## Tag Image
Name componets may contain lowercase letters, digits, and separations. 
To create a new tag for the image that we built:
```
$ docker tag node-docker:lastest node-docker: v1.0.0
```
Now run the `docker images` command to see the list of our local images.
```
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
node-docker         latest              3809733582bc        24 minutes ago      945MB
node-docker         v1.0.0              3809733582bc        24 minutes ago      945MB
node                12.18.1             f5be1883c8e0        2 months ago        918MB
```
Let's removing the tag that we just created. 
```
$ docker rmi node-docker:v1.0.0
untagged: node-docker:v1.0.0
```
# Run Your Image as a Container

## Overview
A container is a normal operating system process except that this process is isolated and has its own file system, its own networking, and its own isolated process tree separate from the host. 
To run an image inside of a container, we use the `docker run` command.
```
$ docker run node-docker
```
When you run this command, you'll notice that you were not returned to the command prompt because our application is a REST server and will run in a loop waiting for incoming requests without return control back to the OS until we stop the container. 
Let's make a GET request to the server using the curl command.
```
$ curl --request POST \
  --url http://localhost:8000/test \
  --header 'content-type: application/json' \
  --data '{
	"msg": "testing"
}'
curl: (7) Failed to connect to localhost port 8000: Connection refused
```
We were not able to connect to localhost on port 8000.

To stop the container, press ctrl-c. 
To publish a port for our container, we'll use the `--publish` flag.
```
$ docker run -p 8000:8000 node-docker
```
Now let's return the curl command from above.
Then, switch back to the terminal where your container is running and you should see the POST request logged to the console.
```
2021-02-11T22:02:17:1830  INFO: POST /test
```
## Run in detached mode
```
docker run -dp 8000:8000 node-docker
```
## List containers
```
docker ps
```
## Stop, Start, and name containers
If we pass the `--all`  or `-a`, we will see all containers on our system whether they are stopped or started.
Locate the name of the container we just stopped and replace the name of the container below in the restart command.
```
$ docker restart <container-id>
```
Stop the container we just started.
```
$ docker stop <container-id>
```
Remove a container 
```
$ docker rm <container -ids>
```
# Use conatiners for development

## Local database and containers
First, we'll take a look at running a database in a container and how we use volumes and networking to persist our data and allow our application to talk with the database.

Let's create our volumes now.
```
$ docker volume create mongodb
$ docker volume create mongodb_config
```

Now we'll create a network that our appliation and database will use to talk with each other. 
```
$ docker network create mongodb
```
Docker will pull the image from Hub and run it for you locally.
```
$ docker run -it --rm -d -v mongodb:/data/db \
  -v mongodb_config:/data/configdb -p 27017:27017 \
  --network mongodb \
  --name mongodb \
  mongo
```
Now, let's update `server.js` to use MongoDB and not an in-momory data store.
```
const ronin     = require( 'ronin-server' )
const mocks     = require( 'ronin-mocks' )
const database  = require( 'ronin-database' )
const server = ronin.server()

database.connect( process.env.CONNECTIONSTRING )
server.use( '/', mocks.server( server.Router(), false, false ) )
server.start()
```
We've add the `ronin-database` module and we updated the code to connect to the databse and set the in-memory flag to false. We now need to rebuild our image so it contains our changes.
```
$ npm install ronin-database
```
Now we can build our image.
```
$ docker build --tag node-docker .
```
We'll need to set the `CONNECTIONSTRING` environment variable so our application knows what connection string to use to access the database. 
```
$ docker run \
  -it --rm -d \
  --network mongodb \
  --name rest-server \
  -p 8000:8000 \
  -e CONNECTIONSTRING=mongodb://mongodb:27017/yoda_notes \
  node-docker
```
Let's test that our application is connected to the database and is able to add a note. 
```
$ curl --request POST \
  --url http://localhost:8000/notes \
  --header 'content-type: application/json' \
  --data '{
"name": "this is a note",
"text": "this is a note that I wanted to take while I was working on writing a blog post.",
"owner": "peter"
}'
```
You should receive the following json back from our service.
```
{"code":"success","payload":[{"name":"this is a note","text":"this is a note that I wanted to take while I was working on writing a blog post.","owner":"peter","id":"8f1642db-1a73-492e-a8d3-814e72f08169","createDate":"2021-02-13T19:05:26.829Z"}]}
```

## Use Compose to develop locally
We'll create a Compose file to start our node-docker and MongoDB with one command.
Open the notes-service in your IDE or text editor and create a new file named `docker-compose.dev.yml`. Copy and paste the below commands into the file
```
version: '3.8'

services:
 notes:
  build:
   context: .
  ports:
   - 8080:8080
   - 9229:9229
  environment:
   - SERVER_PORT=8080
   - CONNECTIONSTRING=mongodb://mongo:27017/notes
  volumes:
   - ./:/code
  command: npm run debug

 mongo:
  image: mongo:4.2.8
  ports:
   - 27017:27017
  volumes:
   - mongodb:/data/db
   - mongodb_config:/data/configdb
volumes:
 mongodb:
 mongodb_config:
 ```
We are exposing `port 9229` so that we can attach a debugger. 
One cool feature of using a Compose file is that we have service resolution set u to use the service name. So we are now able to use `"mongo"` in our connection string. 
To start our application in debug mode, we need to add a line to our `package.json` file to tell npm how to start our application in debug mode.
Open `package.json` file and add the following line to the scripts section:
```
  "debug": "nodemon --inspect=0.0.0.0:9229 server.js"
  ```
  Let's run the following command in a terminal to install nodemon into our project directory.
  ```
  $ npm install nodemon
  ```
  Let's start our aplication and confirm that it is running properly.
  ```
  $ docker-compose -f docker-compose.dev.yml up --build
  ```
  We pass the `--build` flag so Docker will compile our image and then starts it. 

Now let's test our API endpoint. Run the following curl command:
```
$ curl --request GET --url http://localhost:8080/services/m/notes
```
You should receive the following response:
```
{"code":"success","meta":{"total":0,"count":0},"payload":[]}
```

# Run your Tests 

## Running locally and testing the application
Now that we have our sample application locally, let's build our Docker image and make sure everything is running properly.
```
$ docker build -t node-docker .
$ docker run -it --rm --name app -p 8080:80 node-docker
```
Now let's test our application by POSTing a JSON payload and then make an HTTP GET request to make sure our JSON was saved correctly.
```
$ curl --request POST \
  --url http://localhost:8080/services/test \
  --header 'content-type: application/json' \
  --data '{
	"msg": "testing"
}'
```
Now, perform a GET request to the same endpoint to make sure our JSON payload was saved and retrieved correctly.
```
$ curl http://localhost:8080/services/test

{"code":"success","payload":[{"msg":"testing","id":"e88acedb-203d-4a7d-8269-1df6c1377512","createDate":"2020-10-11T23:21:16.378Z"}]}
```

## Install Mocha
Run the following command to install Mocha and add it to the developer dependencies:
```
$ npm install --save-dev mocha
```
## Refactor Dockerfile to run tests
This time, we'lll override the CMD that is inside of our container with npm run test. This will invoke the command that is in the `package.json` file under the "script" section.
```
{
...
  "scripts": {
    "test": "mocha ./**/*.js",
    "start": "nodemon server.js"
  },
...
}
```
Below is the Docker command to start the container and run tests:
```
$ docker run -it --rm --name app -p 8080:80 node-docker npm run test
> node-docker@0.1.0 test /code
> mocha ./**/*.js

sh: 1: mocha: not found
```
  This error is telling us that the Mocha executable could not be found. Let's take a look at the Dockerfile.
  ```
FROM node:14.15.4

WORKDIR /code

COPY package.json package.json
COPY package-lock.json package-lock.json

RUN npm ci --production
COPY . .

CMD [ "node", "server.js" ]
```
The error is occurring because we are passing the `--production` flag to the npm ci command when it installs our dependencies. This tells npm to not install packages that are located under the "devDependencies" section in the `package.json`file. 

### Multi-stage Dockefile for testing
Below is a multi-stage Dockerfile that we will use to build our production image and our test image. 
```
FROM node:14.15.4 as base

WORKDIR /code

COPY package.json package.json
COPY package-lock.json package-lock.json

FROM base as test
RUN npm ci
COPY . .
CMD [ "npm", "run", "test" ]

FROM base as prod
RUN npm ci --production
COPY . .
CMD [ "node", "server.js" ]
```
We first add a label to the `FROM	node:14.15.4` statement. Next we add a new build stage labeled test.
Now let's rebuild our image and run our test.
```
$ docker build -t node-docker --target test .
[+] Building 0.7s (11/11) FINISHED
...
 => => writing image sha256:049b37303e3355f...9b8a954f
 => => naming to docker.io/library/node-docker
```
Now that our test image is built, we can run it in a container and see if our tests pass.
```
$ docker run -it --rm --name app -p 8080:80 node-docker
> node-docker@0.1.0 test /code
> mocha ./**/*.js

  Calculator
    adding
      ✓ should return 4 when adding 2 + 2
      ✓ should return 0 when adding zeros
    subtracting
      ✓ should return 4 when subtracting 4 from 8
      ✓ should return 0 when subtracting 8 from 8

  4 passing (11ms)
```
Update your Dockerfile.
```
FROM node:14.15.4 as base

WORKDIR /code

COPY package.json package.json
COPY package-lock.json package-lock.json

FROM base as test
RUN npm ci
COPY . .
RUN npm run test

FROM base as prod
RUN npm ci --production
COPY . .
CMD [ "node", "server.js" ]
```
Now to run our tests, we just need to run the docker build command as above.
```
$ docker build -t node-docker --target test .
[+] Building 1.2s (13/13) FINISHED
...
 => CACHED [base 2/4] WORKDIR /code
 => CACHED [base 3/4] COPY package.json package.json
 => CACHED [base 4/4] COPY package-lock.json package-lock.json
 => CACHED [test 1/3] RUN npm ci
 => CACHED [test 2/3] COPY . .
 => CACHED [test 3/3] RUN npm run test
 => exporting to image
 => => exporting layers
 => => writing image sha256:bcedeeb7d9dd13d...18ca0a05034ed4dd4
```

# Configure CI/CO
This page guides you through the process of setting up a GitHub Action CI/CD pipleline with Docker containers. 
1. Use a sample Docker project as an example to configure GitHub Actions
2. Set up the GitHub Actions workflow
3. Optimize your workflow to redue the number of pull requests and the total build time
4. Push only specific versions to Docker Hub
## Set up a Docker project
The [SimpleWhaleDemo](https://github.com/usha-mandya/SimpleWhaleDemo) repository contains an Ngnix alpine image. You can either clone this repo or use your own Docker Project.
Before we start, ensure you can access [Docker Hub](https://hub.docker.com/) from any workflows you create. To do this:
1. Add your Docker ID as a secret to GitHub. Navigate to your GitHub repository and click **Settings>Secrets>New secret.**
2.  Create a new secret with the name `DOCKER_HUB_USERNAME` and your Docker ID as value. 
3. Create a new Personal Access Token (PAT). To create a new token, go to Docker Hub Settings and then click **New Access Token**. 
4. Let's call this token **simplewhaleci**.

## Set up the GitHub Action workflow

In the previous section, we created a PAT and added it to GitHub to ensure we can access Docker Hub from any workflow. 
1. The first action enables us to log in to Docker Hub using the secrets we stored in the GitHub Repository.
2. The second one is the build and push action


To set up the workflow:
1. Go to your repository in GitHub and then click **Actions > New Workflow**.
2. Click **set up a workflow yourself** and add the following content:
 
 First, we will name this workflow
 ```
 name: CI to Docker Hub
 ```
 Then, we will choose when we run this workflow.
 ```
 on:
	 push:
			 branches: [main]
```
Now, we need to specify what we actually want to happen within our action.
```
 jobs:
	build:
		runs-on: ubuntu-latest
```
Now, we can ad the steps required. 
```
    steps:

      - name: Check Out Repo 
        uses: actions/checkout@v2

      - name: Login to Docker Hub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKER_HUB_USERNAME }}
          password: ${{ secrets.DOCKER_HUB_ACCESS_TOKEN }}

      - name: Set up Docker Buildx
        id: buildx
        uses: docker/setup-buildx-action@v1

      - name: Build and push
        id: docker_build
        uses: docker/build-push-action@v2
        with:
          context: ./
          file: ./Dockerfile
          push: true
          tags: ushamandya/simplewhale:latest

      - name: Image digest
        run: echo ${{ steps.docker_build.outputs.digest }}
```	

## Optimize the workflow
Next, let's look at how we can optimize the GitHub Actions workflow through build cache. 
1. Build cache reduces the build time as it will not have to re-download all of the images, 
2. it also reduces the number of pulls we complete against Docker Hub. We need to make use of GitHub cache to make use of this. 

First, we need to set up cache for the builder.
```
      - name: Cache Docker layers
        uses: actions/cache@v2
        with:
          path: /tmp/.buildx-cache
          key: ${{ runner.os }}-buildx-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-buildx-
```
And lastly, we need to add some extra attributes to the build and push step.
```
      - name: Login to Docker Hub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKER_HUB_USERNAME }}
          password: ${{ secrets.DOCKER_HUB_ACCESS_TOKEN }}
      - name: Build and push
        id: docker_build
        uses: docker/build-push-action@v2
        with:
          context: ./
          file: ./Dockerfile
          builder: ${{ steps.buildx.outputs.name }}
          push: true
          tags:  ushamandya/simplewhale:latest
          cache-from: type=local,src=/tmp/.buildx-cache
          cache-to: type=local,dest=/tmp/.buildx-cache
      - name: Image digest
        run: echo ${{ steps.docker_build.outputs.digest }}
```
## Push tagged versions to Docker Hub
This involves two steps:
1. Modifying the GitHub workflow to only push commits with specific tags to Docker Hub
2. Setting up a gitHub Actions file to store the latest commit as an image in the GitHub registry

First, let us modify our existing GitHub workflow to only push to Hub if there's a particular tag.
```
on:
  push:
    tags:
      - "v*.*.*"
```
Let's test this:
```
git tag -a v1.0.2
git push origin v1.0.2
```
Now, let's set u a second GitHub action file to store our latest commit as an image in the GitHub regitry.
1. Run your nightly tests or recurring tests,
2. To share work in progress images with collegues

 Let's clone our previous GitHub action and add back in our previous logic for all pushes
 ```
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v1
        with:
        registry: ghcr.io
          username: ${{ github.repository_owner }}
          password: ${{ secrets.GHCR_TOKEN }}
```

Remember to change how the image is tagged:
```
  tags: ghcr.io/${{ github.repository_owner }}/simplewhale:latest
```
# Deploy your app
## Docker and ACI
The Docker Azure Integration enables developers to use native Docker commands to run applications in Azure Container Instance (ACI) when building cloud-native applications. 
## Docker and ECS
The Docker ECS Integration enables developers to use native Docker commands in Docker compose CLI to run applications in Amazon EC2 Container Service(ECS) when buliding cloud-native applications.

