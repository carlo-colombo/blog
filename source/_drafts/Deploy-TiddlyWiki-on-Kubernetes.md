---
title: Deploy TiddlyWiki on Kubernetes
tags:
  - kubernetes
  - tiddlywiki
  - k3s
---

## Discovery of TiddlyWiki

I discovered [TiddlyWiki](https://tiddlywiki.com) reading the [blog of the late Joe Armstrong](https://joearms.github.io/) (of Erlang fame). After reading a post about Erlang and Elixir I started looking into other posts and I found his introduction to TiddlyWiki. The author presented how was using the wiki as his blog and as todo list systems.

I periodically start to investigate, use for a few weeks, and then abandon todo list systems. I went through at least Trello, Google Tasks, Google Keep, textfiles, command line tools, and probably some mobile app. I did not experiment with these systems for a while so I decided to give it a go. As bonus point it can be used personal knowledge base and it seemed infinitly configurable and extendable with JavaScript. A few minutes and I downloaded the one html file that is all that is needed to start use TiddlyWiki.

## My first setup

The original way to use TiddlyWiki is to have a single file that can be resaved when new Tiddlers are added to the wiki. This was not suitable to me as I was planning to access to the wiki from at least: my phone, my laptop and the my work computer. I felt that if I have to manually export the TiddlyWiki then move to another device I would have it abondoned in no time as I would quickly forget, then I would have out of sync wikis and then stopped using it.

TiddlyWiki propose a number of ways to save and share TiddlyWiki, I went with storing it on [Google Drive](https://tiddlywiki.com/static/TiddlyDrive%2520Add-on%2520for%2520Google%2520Drive%2520by%2520Joshua%2520Stubbs.html). It worked and it served me well, but a few times I found my self with out of sync wikis and I have to retrieve the Tiddlers I was interested in from Google Drive revisions. I was able to consume the wiki from both my phone and computers but it has some limitation for concurrent editing, involutary as if it was open on the phone and tried to edit on the computer and it got messy. At the end the backend is a single file that is overriden when the wiki is saved.

In this period I was mostly using it for TODO management using Tiddlers from Joe Armstrong, organizing my job search, and as groceries shopping list. At the same time I started to have an itch to interact programmatically with the wiki(adding, listing, ...).

## Entering k3s

One of the [many option](https://tiddlywiki.com/#GettingStarted) to deploy TiddlyWiki is to run it on [Node.js](https://tiddlywiki.com/#Installing%20TiddlyWiki%20on%20Node.js) that also provide HTTP api to interact with the wiki. After some experimentation on the local machine I was set to deploy it somewhere more persistent and make it available from everywhere.

[k3s](https://k3s.io/) is a lightweight kubernetes distribuition from Rancher that deploys as a single binary and is focused on IoT and edge computing but I think is very suitable to an hobbyst like me that worked on Kubernetes and use something confortable without all the complication of deploying a more featured distribuition. I find it so confortable that I have a k3s node running on my work laptop hosting TiddlyWiki but it could be the argument of another post.

I already maintain a k3s kubernetes cluster (of 1 node) on Scaleway with some Telegram bots and static websites so I started to conjure some yaml to deploy it on the cluster.

After experimenting creating my own container image I find out that `elasticdog/tiddlywiki` exists and it is maintened so I settled with that. The Node.js TiddlyWiki distribution supports authentication and some basic authorization roles (reader and writer), it required me some fiddling with the parameters but at the end I was able to define multiple users (admin, carlo, and bot) with no anonymous access. The cluster is already set for https using certmanager and Let's Encrypt, so I plugged in it with an Ingress controller provided by Traefik and I was able to have it available on the internet reasonably secure with encryption and authentication.

Tiddlywiki need to be pointed to a `wikifolder` that will contain tiddlers and a single configuration file (`tiddlywiki.info`). The proper way to do it would be to run `tiddlywiki <wikifolder> --init` in an init container, make it skip the creation if the file already exist or removing the init container in a following deployment. What I did was more messy as I exec'd in the Tiddlywiki container and then copy the `.info` file created on my machine in the remote volume.

The full configuration is available on [carlo-colombo/tiddlywiki](https://github.com/carlo-colombo/deployment/blob/master/spec/tiddlywiki.yml), I am using [Carvel](https://carvel.dev/) (former k14s) tools ytt, kapp, and kbld to build images, inject secrets and deploy to the cluster (check [`deploy.sh`](https://github.com/carlo-colombo/deployment/blob/master/deploy.sh)).

## Backup

I am using the very good [restic](https://restic.net/) to backup my tiddlers to a remote storage, Scaleway Object Storage (that is S3 compatible). I backup [every 10 minutes](https://github.com/carlo-colombo/deployment/blob/master/spec/backup.yml) because I would be pissed if I was writing something long and lose more than 10 minutes of work, anyway restic is going to only store the diff if any and some metadata so I am not worried about my monthly invoice.

### Conclusion

The setup is quite convuluted and some level of overkill but I feel more confortable using kubernetes and a full declararive deployment than doing it manually with just docker or docker-compose on a server. At the end kubernetes was in my day to day job for a couple of years and I picked a few things along the way. 

Having TiddlyWiki deployed and accessible from anywhere with http available to interact opens up nice possibilities to interact that I am exploring and having fun with it.
