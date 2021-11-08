---
title: Using App Platform with Spring and Angular
description: "Migrating an enterprises application built with  Spring Boot and Angular to the App Platform"
publishDate: {{ now.Format "2006-01-02" }}
lastmod: {{ now.Format "2006-01-02" }}
slug: migrating-springboot-angular-app-to-app-platform
tutorial-articles:
  - Category for the tutorial. Can add multiple entires. For example, kubernetes
---
​
<!--
To have a consistent structure, all tutorials (including READMEs in the https://github.com/digitalocean/container-blueprints repo) must have the following sections:
​
* Title
* Prerequisites 
* Step 1 — Doing the First Thing 
* Step 2 — Doing the Next Thing 
* …
* Step n — Doing the Last Thing 
* Summary
* What’s Next?
​
Style guidelines for each section are described below for quick reference. See https://docs.digitalocean.com/style/tutorial-syle/ for the full Tutorials Style Guide.
-->
​
# Using App Platform with Spring and Angular
​
In this tutorial we want to show how to deploy an enterprise applications on the app plataform. Since we cannot cover every possible case, we focus on a  specific but very common case: an enterprise applications built with Angular for a front-end and Spring for the backend.

For this purpose we selected a well-known open source example application built with with Angular and Spring and ported to App Platform:  this [SpringBoot Angular Shopping Store](https://github.com/zhulinn/SpringBoot-Angular7-Online-Shopping-Store).  You will learn the changes required to deploy it on the App Platform.

You should then be able to easily adapt the steps followed to similar applications built in Javascript for the front-end and Java for the backend as the steps will be very similar. In many cases, the steps will be exactly the same.

<!-- Describes what readers accomplish by following the tutorial. Write the title such that it includes the goal of the tutorial. Examples: 
* Autoscale Cluster With Horizontal Pod Autoscaling
* Transfer DigitalOcean Spaces Between Regions Using Rclone
* Deploy a Sample Web Application Using Terraform
​
<!-- Begin the tutorial with a one to three paragraphs long introduction. Summarize what the tutorial is about, what users will do, create or accomplish, what software is involved, and benefits of using the software configuration. Use phrases such as “you will configure” or “you will build”. 
​
End the introduction with a list of tasks the reader will accomplish by the end of the tutorial. For example:
**In this tutorial, you will:**
1. Download and install Terraform
2. Add your DigitalOcean API Token to Terraform
3. ...
-->
​
## Prerequisites

For local development you need:

- [Node.js](https://nodejs.org/en/download/)
- [Java JDK](https://www.oracle.com/java/technologies/downloads/)
- [Maven](https://maven.apache.org/download.cgi)
- [PostgreSQL](https://www.postgresql.org/download/)
- [Docker Desktop](https://www.docker.com/products/docker-desktop)

Our target application has a frontend written Javscript with the [Angular Framework](https://angular.io), so you need **Node.js** to build it.  In our specific case we verified you need to install  node v14 for your platform. In general you need to install the version of node supported by your version of your javascript framework.

 Note that installing `node` will also install the package management tool `npm` that we also need for the deployment.

Our application has a backend written in Java with the [Spring Framework](https://spring.io/), and, as many other similar applications, it is built using [Maven](https://maven.apache.org/). You need to install Java version 11 for your platform, and, as a separate download, also Maven as it is not bundled  with Java.

The application stores its data in [PostgreSQL](https://www.postgresql.org/). During development you may wanto to run it locally you need a local postgresql, available for all the platforms.

It is not stricly necessary but for testing and debugging purposes you may also need a local Docker. If you use Windows or Mac as your development machine you can use [Docker Desktop](https://www.docker.com/products/docker-desktop).
​
<!--Spells out, in a bulleted list, what the reader should have or do before they follow the current tutorial. Each bullet point must link to existing DigitalOcean documentation or a DigitalOcean tutorial that covers the necessary content if one exists.
​
Be specific with your prerequisites. For example, link to specific concepts or resources instead of “Familiarity with JavaScript”.
-->
​
## Step 1: run the app locally

Let's start first building the application locally.
Fork the [original application](https://github.com/zhulinn/SpringBoot-Angular7-Online-Shopping-Store) in your repository.

I dit and then I cloned it locally as follows:

```
git clone https://github.com/sciabarrado/SpringBoot-Angular7-Online-Shopping-Store
```

The application has a frontend and a backend like many other built as a Single Page Applicaton with an API server.

Let's first build and run the backend.

The backend is a Java application developed with the [Spring Framework](https://spring.io/) framework that uses for storage a [PostgresSQL](https://www.postgresql.org/) database.

You can build an launch the backend with the following commands:

```
cd backend
mvn install
mvn spring-boot:run
```

By default the application is configured to connect to a postgresql databae configured in `backend/src/main/java/resources/application.yml`. If you check that file you will note that it tries by default to connect to a postgres database in localhost named `postgres` using the users `postgres` and the database `root`. 

I tried on my Mac using the [`Posgres.app`](https://postgresapp.com/), a self-container application to run Postgresql and I was lucky as those are the defaults.

So the backend started at the first attempt and created the database.

You can check it connecting to the database showing the tables:

```
$ psql -U postgres -W    
Password: 
psql (14.0)
Type "help" for help.

postgres=# \dt
              List of relations
 Schema |       Name       | Type  |  Owner   
--------+------------------+-------+----------
 public | cart             | table | postgres
 public | order_main       | table | postgres
 public | product_category | table | postgres
 public | product_in_order | table | postgres
 public | product_info     | table | postgres
 public | users            | table | postgres
```

Now let's check and deploy the front-end. 

This is an application built with in JavaScript the [Angular](https://angular.io/) framework.

To install and start it you have to run:

```
cd frontend
npm install
npm start
```

If the prerequisites are satisfied properly you will be able to connect to your browser in post 4020 and [you will see this](1-snapshot.png).

The application is connecting to the backed in port 8080 and it is performing REST requests. You can see this with the command `curl http://localhost:8080/api/product`. It will return a JSON with all the content.

Now we want to deploy this in App Platform. Let's do it.

## Step 2: deploy the front-end as a static site

App Platfrom supports almost out-of the box building the front-end as it is a typical javascript application.

You need to create a `.do/app.yaml file` with a entry for a static site as follows:

```
name: springboot-angular7-store
static_sites:
- name: frontend
  github:
    repo: sciabarrado/SpringBoot-Angular7-Online-Shopping-Store
    branch: main
    deploy_on_push: true
  source_dir: frontend
```

We have to specify where is our source code. So I specified I am using `github`, and then the repo and the branch where is our source. I pushed all the code with the changes in the branch `main`.

Here you can note that we are using the so-called  [monorepo structure](https://docs.digitalocean.com/products/app-platform/how-to/deploy-from-monorepo/), a single repository with multiple components in it. So I am specifying also the subdirectory where is the code of the front-end: `source_dir: frontend`.

Now we have to specify how to build the source code, and where is the result adding:

```
  build_command: ./node_modules/.bin/ng build --prod
  output_dir: dist/shop
```

Note that App Platform build packs will automatically detect it is a node.js application because it will find the `package.json` and will automatically perform an `npm install` that will install all the tools required to build, most notably the CLI `ng` that is the one used by Angular to perform its tasks.

However `ng` is normally installed locally as a global tool. In the build environment it will be avaiable only locally to the front-end build you you will have to use `./node-modules/.bin/ng build --prod` to build. The `.bin` path under the `node_modules` is where `node` install the tools when they are not installed globally. 

Also note that we are specifying the `--prod` flag to build the source in "production" mode. This will use a different path for the API suitable to reach the api on the production server.

Finally the build process will generate the final javascript under the folder `dist/shop` so we have to say to collect and publish the front-end from that directory.

## Step 3: create an appropriate Spring Profile

Spring applications are configured using a configuration file. Our application configuration files are located in   `backend/src/main/java/resources`. The name of the file to use depends on some parameters passed when the application is started. 

If no parametes are passed the name of the configuration file is `application.yml`. If you specifify, for example `--spring.profiles.active=docker` the name of the application file will be `application-docker.yml`. So in short the profile name corresponds to a suffix in the name of the configuration file to use.

So we add the configuration file `src/main/java/resources/application-do.yml` copying the `application.yml` and then modifying it. We will then be able to use it adding at the launch the parameter `--spring.profiles.active=do`.

In order to run the application on App Platform we have to configure at least two things: the database and the base path where the application will be run.

Let's start with the database. In onder to connect to the database we have to specify username, password and the database url where the database is located.

This means we will replace the following entries in our configuration file:

```
 datasource:
    username: ${USERNAME}
    password: ${PASSWORD}
    url: ${JDBC_DATABASE_URL}
```

The syntax `${VARIABLE}` means that the value will be replaced but the environment variable `VARIABLE`. We will provide the actual values for `USERNAME`, `PASSWORD` and `JDBC_DATABASE_URL` using the `app.yaml` when we deploy it (see step 5).

Now, let's examine the base path of our backend.

The frontend will be server at the top level, so invoking `https://<deployment-host>` we will load the angular application and it will start invoking the backend. Now, the frontend is configured to look for t the backend in the same host of the frontend using the prefix `/api`.

When App Platform deploys the backend, you will have to specify under with path it will be served. Since we cannot use the root level (as it is used by the frontend) we are going to use a prefix, that will be `/api`.

However by default the backend is configured to serve every request using the prefix (or, in Java terminology, the context path) `/api`.

This is useful for local development but this will lead in production to require to invoke api calls with the prefix `/api/api` that is not very elegant...

Luckily we can use the configuration file to use an empty context path with the following configuration that we add to our configuation file:

```
server:
  servlet:
    contextPath: /
```

In this way, the backend will serve requests on the root level while App Platforl will add the `/api` prefix, allowing the front-end to invoke the api in the same way it invoked them when doing local development.

## Step 4: build the docker image

Front-end was relatively easy to build, because App Platform build packs supports directly Javascript application. However, App Platform currently supports Java application only using user-provided custom Dockerfiles. Hence we had to write one, changing the existing one.

Unfortnately, the provided Dockerfile assumes you have already built the [jar file](https://docs.oracle.com/javase/tutorial/deployment/jar/basicsindex.html) of your application, that is the executable format of a java application. So you have two options: one is to build and save your jar in your git repository, and another is that you build you jar with Docker.

Since adding binary files in source repositories is not a recommended practice, we go for the longer route of building our app jar using a [Docker multistage build](https://docs.docker.com/develop/develop-images/multistage-build/).

Before we go on discussing the Dockerfile, we note that the source repository includes the `mvnw` script that helps bootstrapping Maven that in turns builds our application. However, for some reasons, it was missing a couple of required files used by this tool, so we had to add manually the following files:

- [backend/.mvn/wrapper/MavenWrapperDownload.java](https://raw.githubusercontent.com/sciabarrado/SpringBoot-Angular7-Online-Shopping-Store/main/backend/.mvn/wrapper/MavenWrapperDownloader.java)
- [backend/.mvn/wrapper/maven-wrappers.properties](https://raw.githubusercontent.com/sciabarrado/SpringBoot-Angular7-Online-Shopping-Store/main/backend/.mvn/wrapper/maven-wrappers.properties)

So you may want to add them manually to the repository to support the multistage build that we are going to explore.

In the following sections we describe the `Dockerfile` that will replace the existing `backend/Dockerfile` in our original application, using a multistage build.

A multistage build in Docker is a Dockerfile that creates multiple containers. Typically some containers are created as a build-only environment, then the artififacts are extracted by other containers that collects the results and prepare the final image to be run.

This is the code of the first part of the `backend/Dockerfile`: 

```
FROM openjdk:11-oracle as builder
USER root
COPY mvnw pom.xml /root/
COPY src /root/src
COPY .mvn /root/.mvn
WORKDIR /root
RUN bash mvnw package -DskipTests 
```

Note we are using as a builder image `openjsk:11-oracle` that includes the Java development kit. Then we copy inside the source of our application, the maven build files (`mvnw` and `pom.xml`) and also the support files required under `.mvn`.

Once those files are in place, building it is just a matter of launching `mvnw` with `bash`. We have to specify that we want to `package` our application and we also need to skip tests as the build environment is lacking the necessary support to actually run them.

Let's see now the second part of our `Dockerfile`:

```
FROM openjdk:11-oracle
WORKDIR /
VOLUME /tmp
COPY --from=builder /root/target/*.jar app.jar
ENTRYPOINT ["java", "-jar","/app.jar", "--spring.profiles.active=do"]
```

Here is is just a matter of copying the jar file built in the previous step as `/app.jar` and launh it, specifying which profile we want to use (in our case the `.do` profile)

## Step 5: deploy the backend

We have now a properly built docker image that connects to the database and can be server under the path that the front-end expects. We have to complete the work adding the configuration for the backend service and the database.

Let's add first a database with the following configuration:

```
databases:
- name: db
  engine: PG
  version: "12"
```

this will deploy a development database postgres when the application will be created. Note that a development database is not recommended for production use. In such a case you may want to provision a full featured DBaaS.

Now it is time to see the backend service. First let's  say how to build and deploy it:

```
- name: backend
  github:
    branch: main
    deploy_on_push: true
    repo: sciabarrado/SpringBoot-Angular7-Online-Shopping-Store
  source_dir: backend
  dockerfile_path: backend/Dockerfile
  http_port: 8080
  routes:
    - path: /api
```

Similarly to the front-end, we are using the sape repo for the backend, but using the monorepo path we specify now the `sorce_dir: backend`. Furthermore we instruct App Platform to build with a dockerfile with `dockerfile_path: backend/Dockerfile`. When using a dockerfile, at the end of the build process the built image will be automatically launched and will start serving, We specified the `http_port: 8080` where the docker container is listening for requests.

As we discussed before, the container itself will be available under the path `/api` that we specify in the `route`.

Now we only miss one thing: passing the invormations to connect to the database. We do that using environment variables. Luckily, App Platform allows every component it deploys to expose some informations that other component can grab. Hence this is the configuration of our service to access to the database:

```
 envs:
  - key: JDBC_DATABASE_URL
      scope: RUN_TIME
      value: ${db.JDBC_DATABASE_URL}
    - key: USERNAME
      scope: RUN_TIME
      value: ${db.USERNAME}
    - key: PASSWORD
      scope: RUN_TIME
      value: ${db.PASSWORD}
```

Note how we are basically propagating to our service as environment variables the values exposed by the database itself. So the `USERNAME` and the `PASSWORD` are the username and password assigned to the database: `db.USERNAME` and `db.PASSWORD`. And, luckily, the database support already provides database urls in the form that the database server expects, so the last one will assign the `JDBC_DATABASE_URL` to `${db.JDBC_DATABASE_URL}`.

<!--Describes what the reader needs to do. Contains commands, code listings, and files, and provides explanations that explain the what and also why the reader is doing it this way.
​
Step title starts with the word Step and a number, followed by a colon. Use gerund, which are -ing words. For example: Step 1: Creating User Accounts.
​
After the title, add an introductory sentence that describes what the reader will do in that step and what role it plays in achieving the overall goal of the tutorial.
​
End each step with a transition sentence that describes what the reader accomplished and where they are going next.
-->
​
## Summary

Let's recap what we learned so far:

- How to build an Angular application leveraging the build tool
- How to configure a Spring Application to connect to a database using environment variables
- How to use multistage docker build to build Java applications using Docker itself
- How to deploy in app platform static sites, databases and services
- How to provide to services informations to connect to databases at runntime
​
<!--Lists what the reader accomplished in a brief workflow format that readers can reference later as a reminder of steps for the task. Include two sections:
​
* Steps that only needed to be done upon initial configuration, such as creating an API token or downloading an application.
* Steps that the reader needs to repeat each time for the solution to work, such as redeploying applications with updated code or running a Terraform job.
​
Instead of using phrases like “we learned how to”, use phrases like “you configured” or “you built”.
-->
​
## What’s Next?

The entire source code of the modified application is [here](https://github.com/sciabarrado/SpringBoot-Angular7-Online-Shopping-Store).


You may want to review the key files and use as a basis for our implementations.

The reference files to review are:

- The final [`app.yaml`](https://github.com/sciabarrado/SpringBoot-Angular7-Online-Shopping-Store/blob/main/.do/app.yaml) able to build and deploy your application.
- The complete [Dockerfile](https://github.com/sciabarrado/SpringBoot-Angular7-Online-Shopping-Store/blob/main/backend/Dockerfile) of the backed to build a Java/Spring application.
- The new [application-do.yaml](https://github.com/sciabarrado/SpringBoot-Angular7-Online-Shopping-Store/blob/main/backend/src/main/resources/application-do.yml) showing what we needed to modify in Spring Application configuration to support accessing App Plaform databases.



<!--Describes what the reader can do next. Can include a description of use cases or features the reader can explore, links to other DigitalOcean tutorials with additional setup or configuration, and external documentation.
-->