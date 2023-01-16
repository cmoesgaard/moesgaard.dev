---
title: Self-hosting
date: 2023-01-16T17:21:32.657Z
---
Over the past few years I've gotten more and more into self-hosting; managing my own server rather than using the publicly available (and often closed-source) apps and services.

My reasons for doing so vary a bit from service to service, but it mostly centers around the following:

* Owning my own data
* Not being in control of what happens to the services I depend on (as when Google Reader shut down a decade ago).
* [And other idealistic open-source related reasons](https://www.gnu.org/philosophy/who-does-that-server-really-serve.html).

## My setup

At the time of writing, I currently run the following services:

* Synapse (a Matrix server for decentralized communication)
* Miniflux (a minimalistic RSS reader)
* Vikunja (an open-source Todoist clone)
* Mealie (a recipe manager)

And the following, to handle the surrounding infrastructure:

* Traefik (open-source edge router)
* Prometheus & friends (for monitoring the server itself)

It's a fairly slim list, but these are just the services I use and depend on in my daily life - I make a habit of checking out new services occasionally.

[Awesome-Selfhosted](https://github.com/awesome-selfhosted/awesome-selfhosted) on Github maintains a pretty comprehensive list of self-hostable alternatives to commercial services.

[/r/selfhosted](https://www.reddit.com/r/selfhosted/) on Reddit is similarly a good source of inspiration.

## Sounds cool, how do I do it?

Self-hosting can be handled in a lot of different ways depending on how ambitious you want to be. Having an interest in running a server *definitely* helps, but isn't strictly required.

I'll explain how my setup is structured.

### A server

I run most of my things off of a single [Hetzner CX21 VPS](https://www.hetzner.com/cloud). 

But for experimentation purposes you can even run the services on your local machine if you wish.

If you want to be able to reach your services outside of your own machine, you'll need a domain name of some kind. Registering a domain and setting up DNS has been left as an exercise to the reader.

### Running the services

Everything is containerized these days, so the most straightforward way of running the various services is through the use of `docker-compose`. Most services will have a `docker-compose.yml` file as part of their documentation or README.md that be used as a template.


