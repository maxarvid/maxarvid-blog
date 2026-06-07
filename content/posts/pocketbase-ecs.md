+++
title = "Hosting Pocketbase on AWS ECS"
date = 2026-06-07
+++

Pocketbase is a single executable written in Go that serves a SQLite-equipped backend. Still in active development, it is nevertheless great for prototyping and having fun, if fun is letting someone else do auth (this is precisely what fun is). I think the point of Pocketbase is that to deploy it you just throw it into a Virtual Private Server (VPS) and forget about it. Avoid over-engineering at all cost. But let's say that for some reason you wanted to host it as a container, with some *personal project*-friendly pricing structure on AWS (looking at you ECS Fargate Spot...) what then?

The first issues you'd need to tackle is persisting anything from the container. Containerized workloads are ephemeral by nature/design, and it's up to us to either mount persistance volumes or hook them up to some type of persitance layer in the form of a database or cache. We can simpulate this locally by defining a Dockerfile in the following fashion:
```Dockerfile
FROM alpine:latest

ARG PB_VERSION=0.39.0
ARG PB_ARCH=linux_amd64

RUN apk add --no-cache \
    unzip \
    ca-certificates

# download and unzip PocketBase
ADD https://github.com/pocketbase/pocketbase/releases/download/v${PB_VERSION}/pocketbase_${PB_VERSION}_${PB_ARCH}.zip /tmp/pb.zip
RUN unzip /tmp/pb.zip -d /pb/

# uncomment to copy the local pb_migrations dir into the image
# COPY ./pb_migrations /pb/pb_migrations

# uncomment to copy the local pb_hooks dir into the image
# COPY ./pb_hooks /pb/pb_hooks

EXPOSE 8080

# start PocketBase
CMD ["/pb/pocketbase", "serve", "--http=0.0.0.0:8080"]
```
Now this is almost verbatum taken from the [Pocketbase docs](https://pocketbase.io/docs/going-to-production/#using-docker) at the time of writing, with a `PB_ARCH` added in because I'm on some Apple processor. To make the local development experience more pleasant, and to prepare for dealing with persistant volumes in our AWS deployment, let's define a Docker Compose file: 
```yml
services:
  pocketbase:
    build: .
    ports:
      - "8080:8080"
    volumes:
      - pb_data:/pb/pb_data
    restart: unless-stopped

volumes:
  pb_data:
```
Yes yes, I know, not strictly required for a single service, but I really like named volumes. When I run `docker volume ls` on my machine, I get pleasant developer-feelings when there are no lengthy hashes I might need to look up. So we build with `docker compose up` and the logs give us the URL to configure our root user. The `/pb/pb_data` is our SQLite data and that is what we need to persist between contaier replacements. So far, so good.

### Going to the Cloud
I hinted above that my motivation for trying this out on AWS Elastic Container Service (ECS) was the Fargate Spot pricing. But where should we slap our persistance volume? At first I wanted to go for Elastic Block Storage (even if it's not multi-Availability Zone) to not have to deal with file locking and Network File System (NFS) limitations. But alas, while EBS if great at very quickly loading in a large amount of data into a task (say you have an LLM or a classic Machine Learning model you want to deploy), it does not persist changes made to said data across container lifespans. 

So we are stuck with an NFS solution in the form of AWS Elastic File System (EFS). If you deploy all this with the Dockerfile above, you will get some angry notes in the logs. Chamge the `CMD` to the below for an immediate and inelegant solution (we can live with this given that we are protoyping, right?):
```dockerfile
CMD ["/pb/pocketbase", "serve", "--http=0.0.0.0:8080", "--logMaxDays=0"]
```
And that's more or less it. I've prepared [this repo](https://github.com/maxarvid/pocketbase-aws-ecs-template) that contains a Cloudformation stack. The stack spins up nearly everything needed (for exceptions, see the REAMDE), and is propbably overkill and not really in the spirit of Pocketbase's raison d'être, but it was fun to pull together and deploy. Hopefully someone (hint: future me) might get some use out of the reasoning.