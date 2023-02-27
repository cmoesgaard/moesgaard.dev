---
title: Self-hosting for fun and profit
date: 2023-02-26T20:41:01.805Z
---
Well, not so much for profit, but it's fun sometimes.

Over the past few years I've gotten more and more into self-hosting; managing my own server rather than using the publicly available (and often closed-source) apps and services.

My reasons for doing so vary a bit from service to service, but it mostly centers around the following:

* Owning my own data.
* Not being in control of what happens to the services I depend on (as when Google Reader shut down a decade ago which I'm somehow still mad about).
* It's a great learning experience.
* [And other idealistic open-source related reasons](https://www.gnu.org/philosophy/who-does-that-server-really-serve.html).

## My setup

At the time of writing, I currently run the following services:

* [Synapse](https://matrix.org/docs/projects/server/synapse) (a Matrix server for decentralized communication).
* [Joplin](https://joplinapp.org/) Server (a synchronization server for Joplin, a note-taking app).
* [Miniflux](https://miniflux.app/) (a minimalistic RSS reader).
* [Vikunja](https://vikunja.io/) (an open-source Todoist clone).
* [Mealie](https://mealie.io/) (a recipe manager).

And the following, to handle the surrounding infrastructure:

* [Traefik](https://traefik.io/traefik/) (open-source edge router).
* [Prometheus](https://prometheus.io/docs/introduction/overview/) & friends (for monitoring the server itself).

It's a fairly slim list, but these are just the services I use and depend on in my daily life - I make a habit of checking out new services occasionally.

[Awesome-Selfhosted](https://github.com/awesome-selfhosted/awesome-selfhosted) on Github maintains a pretty comprehensive list of self-hostable alternatives to commercial services.

[/r/selfhosted](https://www.reddit.com/r/selfhosted/) on Reddit is similarly a good source of inspiration.

## Sounds cool, how do I do it?

Self-hosting can be handled in a lot of different ways depending on how ambitious you want to be. Having an interest in running a server *definitely* helps, but isn't strictly required.

The repository containing my own setup [is open-source](https://github.com/cmoesgaard/home-ops/tree/main/docker-compose). For a long time my setup only lived on the server, so migrating the files to a Git repo is a relatively new thing, so things are still a bit *messy*.

I'll explain a bit, though, about how my setup is structured.

### Server

I run most of my things off of a single [Hetzner CX21 VPS](https://www.hetzner.com/cloud). 

But for experimentation purposes you can even run the services on your local machine if you wish.

In my case, my self-hosting repo has been clumsily checked out on the server manually with `git`🤠 but more elegant solutions can of course be implemented.

If you want to be able to reach your services outside of your own machine, you'll probably need a domain name of some kind. Registering a domain and setting up DNS has been left as an exercise to the reader.

### Running the services

Everything is containerized, so the most straightforward way of running the various services is through the use of `docker-compose`. Most services will have a `docker-compose.yml` file as part of their documentation or `README.md` that be used as a template.

A few collections of `docker-compose` examples have been created and can be used as inspiration, such as [LinuxServer.io](https://github.com/linuxserver) and [Compose-Examples](https://github.com/Haxxnet/Compose-Examples).

The `docker-compose.yml` files you find can often be used as-is but I've done the following for mine:

* All data persisted in [Docker volumes](https://docs.docker.com/storage/volumes/) rather than bind mounts.
* Each service stack is in its own `docker-compose.yml` file. This helps ensure that a private network is created for each service, restricting access to your various databases. This also allows me to easily stop/start individual services.

### Traefik

Traefik helps tie the whole thing together, and makes everything reachable from the outside. Traefik is an edge router - essentially a reverse proxy - responsible for routing traffic from the outside to the various services running on your server.

It also automatically takes care of issuing TLS certificates via LetsEncrypt.

Traefik again runs as its own container, which is the only container with port-bindings to the host machine.

Traefik supports dynamic [configuration discovery](https://doc.traefik.io/traefik/providers/docker/), which in our case allows us to specify the relevant configuration as labels on the individual Docker containers: 

```yaml
version: "3.1"
services:
  mealie:
    image: hkotel/mealie
    container_name: mealie
    restart: unless-stopped
    depends_on:
      - "postgres"
    environment:
      DB_ENGINE: postgres
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_SERVER: postgres
      POSTGRES_PORT: 5432
      POSTGRES_DB: mealie
    networks:
      - traefik
      - mealie
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.mealie.rule=Host(`${HOSTNAME}`)"
      - "traefik.http.routers.mealie.entrypoints=websecure"
      - "traefik.http.routers.mealie.tls.certresolver=mytlschallenge"
      - "traefik.docker.network=traefik"
  postgres:
    container_name: postgres
    image: postgres:14
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_USER: ${DB_USER}
    networks:
      - mealie
    volumes:
      - db:/var/lib/postgresql/data/

networks:
    traefik:
        external: true
    mealie:

volumes:
  db:
```

In this example, we have a Mealie app with corresponding database. The app serving the frontend has been given labels describing how it should be reachable from the outside. Traefik will automatically read the configuration from the labels on container startup and register the appropriate routes (and tear them down on container shutdown).

As previously mentioned, I have a separate `docker-compose.yml` file per service, so I've registered an external `traefik` docker network that is shared by Traefik and every service I want to route data to.

## What's next?

Despite its shortcomings, I've deemed my own setup Good Enough™ and it has served me pretty well for a number of years so far. But there's always room for improvement.

In the future I'd like to improve on a few things, each of which probably deserve their own posts:

* Open-sourcing my infrastructure. Cleaning up my repo, and making what I have available for others as inspiration.
* A way to automatically update my services (probably using [Watchtower](https://containrrr.dev/watchtower/)).
* A more robust observability setup.
* A GitOps-y way to have changes to my repository automatically be reflected on the server, so I don't have to SSH out and do stuff with my hands.
* Potentially migrate my setup to Kubernetes, starting with a k3s cluster. This will at least allow me to easily implement a GitOps workflow with FluxCD or similar, and will serve as a ⚰️fun⚰️ learning experience.

  * [k8s-at-home](https://k8s-at-home.com/) is a community focused on doing exactly this, and they have a lot of ready-made Helm charts for many services.