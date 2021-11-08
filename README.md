---
title: Using App Platform with Spring and Angular
description: "How to migrate step-by-step an enterprises application built with Spring Boot and Angular to the App Platform"
publishDate: {{ now.Format "2006-01-02" }}
lastmod: {{ now.Format "2006-01-02" }}
slug: migrating-springboot-angular-app-to-app-platform
tutorial-articles:
  - AppPlatform 
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
In this tutorial we will teach how to deploy an enterprise applications on the App Platform. Since we cannot cover every possible case, we focus on a  specific but very common case: an applications built with Angular for a frontend and Spring for the backend.

For this purpose we selected a well-known open source example application, built with with Angular and Spring, and ported to App Platform: this [SpringBoot Angular Shopping Store](https://github.com/zhulinn/SpringBoot-Angular7-Online-Shopping-Store). You will learn the changes required to deploy it on the App Platform. We expect those changes are very close to those required for any application built with the same technologies.

You should then be able to easily adapt the steps described to your applications built in Javascript for the frontend and Java for the backend. In many cases, the steps will be exactly the same.

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

Since target application has a frontend written Javscript with the [Angular Framework](https://angular.io), you need **Node.js** to build it. In our specific case we verified you need to install node v14 for your platform. In general you need to install the version of node supported by your version of your javascript framework.

 Note that installing `node` will also install the node package manager `npm`, that we also need for the deployment.

Our application has a backend written in Java with the [Spring Framework](https://spring.io/), and, as many other similar applications, it is built using [Maven](https://maven.apache.org/). You need to install Java version 11 for your platform. You also need to install Maven as a separate download, as it is not bundled with Java.

The application stores its data in SQL databases, and defaults to[PostgreSQL](https://www.postgresql.org/). During development you may wanto to run it locally you need a local postgresql. Also PostgreSQL is available for all the platforms.

It is not stricly necessary but for testing and debugging purposes you may also need a local Docker.
If you use Windows or Mac as your development machine you can use [Docker Desktop](https://www.docker.com/products/docker-desktop).
​
<!--Spells out, in a bulleted list, what the reader should have or do before they follow the current tutorial. Each bullet point must link to existing DigitalOcean documentation or a DigitalOcean tutorial that covers the necessary content if one exists.
​
Be specific with your prerequisites. For example, link to specific concepts or resources instead of “Familiarity with JavaScript”.
-->
​
## Step 1: run the app locally

Let's start building the application locally, forking the [original application](https://github.com/zhulinn/SpringBoot-Angular7-Online-Shopping-Store) in your repository.

Once we forked it, we cloned the source locally as follows:

```
git clone https://github.com/sciabarrado/SpringBoot-Angular7-Online-Shopping-Store
```

The architectue of the application is a Single Page Applicaton connecting to a RESTful API server. Let's run it, first building and launching the backend, with the following commands:


```
cd backend
mvn install
mvn spring-boot:run
```

By default, the application is configured to connect to a local postgresql database configured in `backend/src/main/java/resources/application.yml`. If you check the configuration you will see  it uses a postgres database server in localhost using the default database and user.

I tried on my Mac using the [`Posgres.app`](https://postgresapp.com/), a self-container application to run Postgresql and I was lucky as it worked without any change to the database.

We checked everything worked fine verifying the database tables are created properly, as follows: 

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

You can also verify the API server is working properly with this with the command:

```
curl http://localhost:8080/api/product
``` 

It should return a non-empty JSON with a catalog of products. 

Now let's build the frontend.  To install and start it you have to run:

```
cd frontend
npm install
npm start
```

If you have installed the correct prerequisites you will be able to connect to your browser in post 4020 and [you will see this](1-snapshot.png).

The application is connecting to the backed in port 8080 and it is performing REST requests. 

So far so good. The application is properly running locally, and we learned how to build it. Now we want to deploy this in App Platform, and we will need to do some changes to adapt to the production environment.

## Step 2: deploy the front-end as a static site

App Platfrom supports almost out of the box building the frontend, as it is a common javascript application.

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

First, we have to specify where is our source code. As it is stored on GitHub, we specified `github`, then the repo and the branch where is our source. 

Note that we pushed the code with our changes in the branch `main` (the original branch of the example is `master`).

Here you can note that we are using the so-called  [monorepo structure](https://docs.digitalocean.com/products/app-platform/how-to/deploy-from-monorepo/): a single repository with multiple components in it. 

So we have to tell to App Platform the component is not on the root of the repository but in a subdirectory:  `source_dir: frontend`.

Also we have to specify how to build the source code, and where is the result:

```
  build_command: ./node_modules/.bin/ng build --prod
  output_dir: dist/shop
```

Note that App Platform buildpack will automatically detect it is a node.js application because it will find the `package.json` and will automatically perform an `npm install`. This will installs all the tools required to build the frontend; most notably it installs the CLI `ng` that is the tool required to build any angular application.

However `ng` is normally installed locally as a global tool. In the build environment it will be avaiable only under `node_modules`.  So you will have to use `./node-modules/.bin/ng build --prod` to build. 

The `.bin` path under the `node_modules` is where `node` install the tools when they are not installed globally.  Also note we need to add the  the `--prod` flag to build the source in "production" mode. This will use a different path for the API suitable to reach the api on the production server. 

Finally the build process will generate the final javascript under the folder `dist/shop`; so we have to say to collect and publish the frontend from that directory.

## Step 3: create an appropriate Spring Profile

Spring applications are configured using a configuration file, located in our case under   `backend/src/main/java/resources`. 

The name of the configuration file  depends on some parameters passed when the application is started.  If no parametes are passed the name of the configuration file is `application.yml`. If you specifify, for example `--spring.profiles.active=docker` the name of the configuration file will be `application-docker.yml`.  The profile name corresponds to a suffix in the name of the configuration file to use.

So we are going to add a new configuration file `src/main/java/resources/application-do.yml` copying the `application.yml` and then modifying it. We will then be able to use it adding at the launch the parameter `--spring.profiles.active=do`.

In order to run the application on App Platform we have to configure at least two different components: the database and the route to reach backend service.

Let's start with the database. You connect to the database specifying username, password and the database url where the database is located.

This means we will replace the following entries in our configuration file:

```
 datasource:
    username: ${USERNAME}
    password: ${PASSWORD}
    url: ${JDBC_DATABASE_URL}
```

In general the syntax `${VARIABLE}` means that the value will be provided by the environment variable `VARIABLE`. We will pass  the actual values for `USERNAME`, `PASSWORD` and `JDBC_DATABASE_URL` using the `app.yaml` when we deploy it (see step 5).

Now, let's examine the route to reach our backend. App Platform provides a `<deployment-host>` hostname to access all the components of our application via https. The various compoments are then accessible using different subpaths connecting to that hostname. 

The frontend will served at the root level, so invoking `https://<deployment-host>/` we will open the angular application. Once it is loaded in the browser, d it will start invoking requests on the backend. Now, the frontend is configured to look for the backend in the same host using the prefix `/api`. Hence the backed must be accessible as `https://<deployment-host>/api/`.

When App Platform deploys the the services, you will have to specify for each one under which path it will be served. 

Note however that by default the backend is configured to serve every request using the prefix (or, in Java terminology, the context path) `/api`.

This is useful for local development but this will lead in production to require to invoke api calls with the prefix `/api/api` that is not very elegant, and will not work unless we change the frontend configuration.

Luckily, we can use the configuration file to use an empty context path. We need to use  following configuration in our configuration file:

```
server:
  servlet:
    contextPath: /
```

In this way, the backend service will serve requests on its root level, while App Platform will add the `/api` prefix, allowing the frontend to invoke the api in the same way it invoked them when doing local development. And everything will work in production.

## Step 4: build the docker image

Front-end was relatively easy to build, because App Platform buildpack supports directly Javascript application. However, App Platform currently supports Java application only using user-provided custom Dockerfiles. Hence we had to write one.


Unfortnately, the provided Dockerfile assumes you have already built the [jar file](https://docs.oracle.com/javase/tutorial/deployment/jar/basicsindex.html) of your application, that is the executable format of a java application. 

So we have two options: one is to build and save our jar in the repository, and another is that you build the jar with Docker.

Since adding binary files in source repositories is definitely not a recommended practice, we go for the longer route of building our app jar using a [Docker multistage build](https://docs.docker.com/develop/develop-images/multistage-build/).

*NOTE* Before we go on discussing the Dockerfile, we note that the source repository includes the `mvnw` script that  bootstraps Maven to build our application. However, for some reasons, a couple of  required files are missing, so we had to add manually the following files:

- [backend/.mvn/wrapper/MavenWrapperDownload.java](https://raw.githubusercontent.com/sciabarrado/SpringBoot-Angular7-Online-Shopping-Store/main/backend/.mvn/wrapper/MavenWrapperDownloader.java)
- [backend/.mvn/wrapper/maven-wrappers.properties](https://raw.githubusercontent.com/sciabarrado/SpringBoot-Angular7-Online-Shopping-Store/main/backend/.mvn/wrapper/maven-wrappers.properties)

If you are going to repeat the steps of this tutorila, you may want to add them manually.

Now, let's describe our the `backend/Dockerfile` for App Platform. It will replace the existing `backend/Dockerfile` in our original application, using a multistage build.

A multistage build in Docker is a Dockerfile that creates multiple containers. Typically some containers are created as a build-only environment, then the produced artifacts are copied in other containers to build the final image to be run.

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

Note we are using as a builder image `openjdk:11-oracle` that includes the Java development kit. Then we copy inside the source of our application, the maven build files (`mvnw` and `pom.xml`) and also the support files required under `.mvn`.

Once those files are in place, building it is just a matter of launching `mvnw` with `bash`. We have to specify that we want to `package` our application and we also need to skip tests, as the build environment is lacking the necessary database support to actually run them.

Let's see now the second part of our `Dockerfile`:

```
FROM openjdk:11-oracle
WORKDIR /
VOLUME /tmp
COPY --from=builder /root/target/*.jar app.jar
ENTRYPOINT ["java", "-jar","/app.jar", "--spring.profiles.active=do"]
```

Here is is just a matter of copying the jar file built in the previous step as `/app.jar` and lauching it, specifying which profile we want to use (in our case the `.do` profile). Remember we wrote in the previous step a `application-do.yml` configuration to support the App Platform environment.

## Step 5: deploy the backend

We have now a properly built docker image that connects to the database and can be served under the path that the frontend expects. 

We have to complete the work adding to the `app.yaml` the configuration for the backend service and the database.

Let's add first a database with the following configuration:

```
databases:
- name: db
  engine: PG
  version: "12"
```

This will deploy a Postgres development database  when the application will be deployed in App Platfor. Note that a development database is not recommended for production use. In such a case you may want to provision a full featured database using one of the offerings of the Digital Ocean's DBaaS.

Now it is time to see deploy backend service. First let's  add the configuration to build and deploy it:

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

Similarly to the frontend, we are using the same repo for the backend; but because we are using the monorepo pattern, we have to specify now we wat to build the backend component, with  `source_dir: backend`. 

Furthermore we instruct App Platform to build with a dockerfile (and not with a build pack) with `dockerfile_path: backend/Dockerfile`. 

When using a dockerfile, at the end of the build process the built image will be automatically launched and will start serving. We say that the port to use is the `http_port: 8080`.

As we discussed before, the container itself will be available under the path `/api` that we specify in the `route`.

Now we only miss one thing: passing the informations required to let the backend to connect to the database. We do that using environment variables. 

Luckily, App Platform allows every component it deploys to expose some informations that other component can read. Hence we add to the configuration of our service the following environment variables to access to the database:

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

We are ready. It you now run 

```
doctl app create --spec .do/app.yaml
```
​
after a while you shold see the application up and running exposing the same user interface you have see at the beginning, when testing the application locally.

## Summary

Let's recap what we learned so far:

- how to build an Angular application leveraging the build `ng` tool
- how to configure a Spring Application to connect to a database using environment variables
- how to use multistage docker build to build Java applications using Docker itself
- how to deploy in app platform static sites, databases and services
- how to provide to services informations to connect to databases at runtime
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

You are now ready to deploy your Spring/Angular enterprise application to App Platform.

<!--Describes what the reader can do next. Can include a description of use cases or features the reader can explore, links to other DigitalOcean tutorials with additional setup or configuration, and external documentation.
-->