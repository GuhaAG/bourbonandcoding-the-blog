---
title: Dockerize a React app and deploy it on an AWS EC2 instance
---

### Why Docker ?

You don't _need_ to dockerize your app to deploy it on AWS. So why docker ? There are many reasons to use docker containers and I won't cover everything here, but personally why i would containerize any app is dependency management. Modern web applications come with loads of dependencies, and having to install everything on every environment you want to run it on, or worse yet, run it on a shared environment with other apps perhaps requiring other versions of the same libraries, is complicated. With your app residing on a docker image all you do is pull the image and run it, docker handles the rest.

### Why EC2

No reason really. The steps in this post can be used to dockerize your app and then run the image on any machine you want, on the cloud or otherwise. I use ec2 in this article because it is easy and familiar, and *free* (within the free tier limit).

### Create React app
For this article i am going to use a boilerplate react app using facebook's `create-react-app`.

```sh
npx create-react-app react-docker-example
cd react-docker-example && npm install
npm start
```

Check out your brand spanking new React web app at [http://localhost:3000/]

### Docker
Now let's create a `Dockerfile` in the root of the app.

```sh
# build stage
FROM node:lts-alpine as build-stage
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build     
    
# production stage
FROM nginx:stable-alpine as production-stage
COPY --from=build-stage /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
    
- The `FROM` image is the base image we use to run our stages. In the build stage it is a node js image since we need node js to build a react app, and in the production stage we use nginx to serve the app.
-  In the build stage we save the build artifact and then use only that in the production stage, this saves a lot of space in our image.
- We serve the build artifact with nginx at a port of our choosing.


Now let's run it locally to see if it works.
First we build our docker image. 

```sh
docker build -t bourbonandcoding/react-docker-example .
#             ^                  ^                    ^
#            tag              tag name       Dockerfile location
```

Now we run it

```sh
docker run -p 3000:80 -d bourbonandcoding/react-docker-example:latest
#              ^       ^                       ^    
#              |    detached mode          tag name    
#       host machine port : docker port 
```

- Detached mode, shown by the option `--detach` or `-d`, means that a Docker container runs in the background. It does not receive input or display output.

Now your React app should be available again at [http://localhost:3000/]
> `Image v Container` : A running instance of an image is called a container. If you start an image, you have a running container of this image. You can have many running containers of the same image. 
> You can see all your images with  `docker images`  whereas you can see your running containers with  `docker ps`  (and you can see all containers with  `docker ps -a`).

Next, we push the docker image to a repository. Let's use a `dockerhub` public repository.
You need to `docker login` first with your user/pass and create a public repository. We will be pushing our image to this repository.

Let's check the image ID first

```sh
docker images
```

We get a list of our images and their IDs

```sh
REPOSITORY                              TAG         IMAGE ID 
bourbonandcoding/react-docker-example   latest      bf3e546c6845
```

Next we tag the image 

```sh
docker tag bf3e546c6845 <dockerhub-username-here>/bourbonandcoding:v1
```

- \<dockerhub-username-here\>/bourbonandcoding is the name of my dockerhub public repository here. `v1` is the tag.

Now we can push it to our dockerhub public repository

```sh
docker push <dockerhub-username-here>/bourbonandcoding:v1
```

Now the image is pushed to a public repository accessible to everyone. We are going to be pulling it to our ec2 instance next.

>If you don't want to share your image with the world, you can also use private repositories. You get 1 private repository free with your dockerhub account and can pay for more.

### Deploy on EC2
For starters, i will assume you have an aws account and have launched and started an ec2 instance, sshed into it and installed docker if necessary.

Pull the previously created image from `dockerhub`.

```sh
docker pull <dockerhub-username-here>/bourbonandcoding:v1
```

Then, run it 

```sh
docker run -p 80:80 -d <dockerhub-username-here>/bourbonandcoding:v1
```
That's it, since we bound it to port `80` the app should be running on the public IP of the instance now. 
> If `localhost` is coded into your server configuration, you have to change it to `0.0.0.0` to bind it to the public IP.

#### Next Steps
If you want to share you shiny new web app with the world wide web, you would want to get a static IP for your instance, allow TCP connections to it by changing the security group configuration and perhaps even get a domain name and associate it to the IP. I won't cover any of that in this post but buy me a bourbon or two and i might be convinced to make another post all about that. 

Until then cheers, and keep coding !

Find the code used in this post [here](https://github.com/GuhaAG/bourbonandcoding/tree/master/react-docker-example).