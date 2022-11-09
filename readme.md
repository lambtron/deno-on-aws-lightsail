# Deploy Deno to Amazon Lightsail

Amazon Lightsail is the easiest and cheapest way to get started with Amazon Web Services. It allows you to host virtual machines and even entire container services.

This How To guide will show you how to deploy a Deno app to Amazon Lightsail using Docker, Docker Hub, and GitHub Actions.

Before continuing, make sure you have:
- [`docker` CLI](https://docs.docker.com/engine/reference/commandline/cli/)
- a [Docker Hub account](https://hub.docker.com)
- a [GitHub account](https://github.com)
- an [AWS account](https://aws.amazon.com/)

## Create Dockerfile and docker-compose.yml

To focus on the deployment, our app will simply be a `main.ts` file that returns
a string as an HTTP response:

```ts
import { Application } from "https://deno.land/x/oak/mod.ts";

const app = new Application();

app.use((ctx) => {
  ctx.response.body = "Hello from Deno and AWS Lightsail!";
});

await app.listen({ port: 8000 });
```

Then, we'll create two files -- `Dockerfile` and `docker-compose.yml` -- to
build the Docker image.

In our `Dockerfile`, let's add:

```Dockerfile
FROM denoland/deno

EXPOSE 8000

WORKDIR /app

ADD . /app

RUN deno cache main.ts

CMD ["run", "--allow-net", "main.ts"]
```

Then, in our `docker-compose.yml`:

```yml
version: '3'

services:
  web:
    build: .
    container_name: deno-container
    image: deno-image
    ports:
      - "8000:8000"
```

Let's test this locally by running `docker compose up` and going to
`localhost:8000`.

![hello world from localhost](/static/hello-world-from-localhost.png)

It works!

## Build, Tag, and Push to Docker Hub

First, let's sign into [Docker Hub](https://hub.docker.com/repositories) and create a repository. Let's name it `deno-on-aws-lightsail`.

Then, let's tag and push our new image, replacing `username` with yours:

```
docker compose -f docker-compose.yml build
docker tag deno-image {{ username }}/deno-on-aws-lightsail
docker push {{ username }}/deno-on-aws-lightsail
```

After that succeeds, you should be able to see the new image on your Docker Hub repository:

![new image on docker hub](/static/new-image-on-docker-hub.png)

## Create and Deploy to a Lightsail Container

Let's head over to [the Amazon Lightsail console](https://lightsail.aws.amazon.com/ls/webapp/home/container-services).

Then click "Containers" and "Create container service". Half way down the page, click "Setup your first Deployment" and select "Specify a custom deployment".

You can write whatever container name you'd like.

In `Image`, be sure to use `{{ username }}/{{ image }}` that you have set in your Docker Hub. For this example, it is `lambtron/deno-on-aws-lightsail`.

Let's click `Add open ports` and add `8000`.

Finally, under `PUBLIC ENDPOINT`, select the container name that you just created.

The full form should look like below:

![create container service interface](/static/create-container-service-on-aws.png)

When you're ready, click "Create container service".

After a few moments, your new container should be deployed. Click on the public address and you should see your Deno app:

![Hello world from Deno and AWS Lightsail](/static/hello-world-from-deno-and-aws-lightsail.png)

## Automate using GitHub Actions

In order to automate that process, we'll use the `aws` CLI with the [`lightsail` subcommand](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/lightsail/push-container-image.html).

The steps in our GitHub Actions workflow will be:
1. Checkout the repo
2. Build and Push our app as a Docker Image
3. Install and configure AWS CLI
4. Push image from Docker Hub to AWS Lightsail Container Service

Follow [this AWS guide](https://lightsail.aws.amazon.com/ls/docs/en_us/articles/lightsail-how-to-set-up-access-keys-to-use-sdk-api-cli) to get generate an `AWS_ACCESS_KEY_ID` and `AWS_SUCCESS_ACCESS_KEY`. _[See here to learn more about managing access to Amazon Lightsail for an IAM user.](https://github.com/awsdocs/amazon-lightsail-developer-guide/blob/master/doc_source/amazon-lightsail-managing-access-for-an-iam-user.md)_

