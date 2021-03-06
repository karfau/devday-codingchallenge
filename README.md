# DEV DAY 19 Coding Challenge

Goal: build a website that aggregates meaningful content for developers. The way to go: build tiny services that respond with pieces of content and stitch them together using a monolithic frontend application. While we added Dockerfiles to enable you to run everything locally we recommend to use [platform.sh](platform.sh)'s great multi-environment PaaS to contribute. 

# Requirements
## the **absolutely minimal** requirements
- git
- [the platformsh client](https://docs.platform.sh/gettingstarted/cli.html) (requires php)
- an account with access [to our platformsh project](https://console.platform.sh/robertdouglass/ulyecw4ca3wk6)
- a great [code editor](https://code.visualstudio.com/) 

with that setup you can write code locally, fork and push to your very own platform.sh environment and run it there. 

## the **minimal** requirements
It's taking some time until platform.sh has built and released your code, therefore we suggest to at least install your micro application's language stack locally. The machines we provide during DEV DAY 19 are provisioned with Ruby, Python, node.js and PHP, composer and the platform.sh cli but also Java and Go are supported. You don't necessarily have to install the resource dependencies (like MySQL, Redis or Elasticsearch) locally because platform.sh allows you to connect to your environments' resources via an SSH tunnel. Since some tools we've built to make the challenge less challenging rely on node.js, we recommend that you at least install node:

- [nodejs](https://nodejs.org/en/) (if you want to avoid littering your local machine set it up using [nvm](https://github.com/nvm-sh/nvm))

If you want to work on the **frontend code** (the `scaffolding` app), checkout the README.md in the `scaffolding` directory. You'll only need node / yarn and an environment with a working coordinator for this (you can use the master environment on https://coordinator.devday.tk).

# Getting started 
Follow the instructions on platform.sh's [Getting Started](https://docs.platform.sh/gettingstarted/tools.html) page. They boil down to authenticating with platform.sh using the CLI client (`platform`) and adding your public SSH key to the project (`platform ssh-key:add`).

Get the project's sources from platform.sh (it's a `git clone` under the hood)

`platform get <platform project id>`

The default branch that's checked out is unsurprisingly `master`. You should use that as starting point for your own environment. You can fork one with

`platform branch <a good name>`

which creates a branch, pushes it to platform.sh and triggers a deployment of a new environment. When forking an environment, platform.sh creates a **complete** (!) clone of it, all its apps and its dependencies and exposes it as new environment to the web. 

You can now cd into an application and start adding features to it or create a new one. Every time you `platform push` new commits your environment will be updated.

### Access the running applications
`platform url` displays a list of available endpoints. There are 2 fundamental and 2 demo apps available as of May 24th:

- **coordinator** is responsible for "coordinating" the microservices with the frontend code (and with each other). It's based on node.js / express.
- **scaffold** is a React based monolithic frontend application that requests information about all available microservices from **coordinator** . It's based on Create React App / Yarn.

- **rssreader** reads an rss feed every 15 minutes, stores its items in an ElasticSearch index and responds with the latest news. It's built on node.js / zeit/micro
- **calendarservice** is a dumb (your turn!) service that yields a list of days of the current month. It's written in PHP using Symfony 4.2

You can access all of these applications on their subdomains of the same name on your environment url.

### Make a change and deploy
Lets try to change an existing service. First, create a new environment:

`platform branch messagerss` (use any branch name you like)

While platform.sh's builder building your clone of the master branch you can already start hacking:

- open `rssreader/index.js`
- add another field in its output (e.g. a string message): 

```js
const response = {
    message: "Pray to the Demo Gods",
    news: result.body.hits.hits.map(h => h._source)
  };
``` 

and commit it (use your IDE or `git commit -am "added message"`). Now you're ready to push that change with `platform push`. Platform.sh will only rebuild the rssreader app which in some seconds. With `platform url` you can display the individual URLs of your environment and access your version of the rssreader application. 

**NOTICE** *if you see a big fat security warning, that's because LetsEncrypt certificates need some time to propagate. Try to find the "Accept the risk" button (it's your code!) and proceed.

### Run an application locally with tethered resources
If you try to run the rssreader application locally (`npm run start`) it will fail since you most likely don't have its Elasticsearch resource dependency installed. The good news: you can simply connect your locally built application to the Elasticsearch instance on your platform.sh environment. 

```bash
platform tunnel:open && export PLATFORM_RELATIONSHIPS="$(platform tunnel:info --encode)"
```

Run `npm run start` again and you'll see the output that you expect. The mechanisms of how environment variables are exposed to services are explained in detail [in the platform.sh docs](https://docs.platform.sh/gettingstarted/local/tethered.html#quick-start).

### Creating a new application
Platform.sh relies on routing- , services-, and local file descriptions that define how an application's services are built, mounted to the request gateway and which resources it relies on. You can find all the details [in their docs](https://docs.platform.sh/overview/structure.html) but to get started quickly, we've built a bash script that creates a minimal set for a new application right away. 

Create a new environment (see above), change to the project root and run

`./new.sh fortune`

That should give you a `fortune` directory containing a `platform.app.yaml`, a new "fortune" route in the `.platform` directory's routing descriptor and docker and proxy definitions (you can safely ignore (but commit) them - they're **only** needed for a completely local setup).

Now it's up to you to decide your favorite language and checkout the samples over at platform.sh's decent [language samples repository](https://github.com/platformsh/language-examples). Look into the local `platform.app.yaml` files to define how your application is supposed to run. 

### Going live / merging upstream
Once you're satisfied with the results you can ask the project maintainer to merge your environment into master (`platform merge`) and start the process again. You can remove an environment with `platform environment:delete` or switch back to master `platform environment:checkout master`. 

# Build all services locally
In the unlikely case that you have installed all language dependencies on your local machine you can follow the instructions:

```bash
cd dev-day-coding-challenge
platform build
``` 

This will install dependencies and build all applications and microservices **locally** on your machine and symlink them into a new `_www` folder. NOTICE: Symfony's base requirements quite likely already break this default build (since it might miss the php-xml package, that you can install on Ubuntu with `apt-get install php-xml`). If you want, you can start fixing that, but it's much nicer to run them on Docker instead.

## Run everything locally with Docker
**NOTICE** *Here be dragons. If you just want to hack on the microservices or the frontend, you can safely ignore all of the .env, localEnv, Dockerfile, docker-compose.yml files and the web folder (it's a local reverse proxy).*

Since we're going to build a lot of small applications that will be bound together in a monolithic frontend, it's not really feasible to launch them one by one, remember ports and configure them accordingly. That's why we have prepared a docker/compose environment that should enable you to run **everything** locally inside Docker containers. You'll also need a local *node* installation from here on. 

- On Linux you can't simply mount a service on port 80 without root implications, so first copy the `.env.dist` file in the project root to an `.env` file and change the port (or leave it as 8000)
- `make prepare` creates environment files in the coordinator directory. By default everything will run on `devday.local`. If you want to change that, edit the generated `coordinator/.env` file's `DEFAULT_HOST` entry now. 
- `make coordinator` does quite frankly what platform.sh is doing remotely: it scans directories for platform.app.yamls and generates routing information available to all other applications. We're exposing that (as platform.sh does) as b64 encoded env var `$PLATFORM_ROUTES`
- `make start` pulls images, installs dependencies and starts all services by calling `docker-compose up`. On a fresh setup this takes around 10 minutes.
- add localhost aliases for all services to your `/etc/hosts` file. At the time of writing this would look like:
```text
127.0.0.1 devday.local
127.0.0.1 coordinator.devday.local
127.0.0.1 scaffold.devday.local
127.0.0.1 rssreader.devday.local
```

if everything goes well, you should be able to visit `http://devday.local:8000` (whatever port you chose) and see the freshly built frontend consuming Docker based microservices as exposed by your local coordinator service (`coordinator.devday.local:8000`)

#### Some Docker hints
- `docker-compose exec <container> sh`: get a shell on a running container
- `docker-compose restart <container>`
- `docker-compose logs <container>`
make sure to also check the `Makefile` for more insights.

# Frontend development
please check the README.md in `./scaffold`

# Troubleshooting

## I change code but nothing changes
Are you coming from a PHP background? Node.js and many other languages basically need to be restarted to reflect code changes (some frameworks like zeit/micro-dev can do that automatically). When you run in Docker, you can restart them with `docker-compose restart <container>`

## There are no news in the rss reader 
We're storing all news that have been fetched from an RSS feed in Elasticsearch. On platform.sh that happens through cron. On your local environment you're on your own :) Execute the `rssreader/bin/readFeeds.js` file to populate the database. Make sure to copy the .env.dist file to a local .env file. You can also execute it directly in your environment's instance by sshing into it (`platform ssh`)

#### Beware of CORS!
Since we're connecting all microservices from the client side, they must send appropriate CORS headers on every request. This cannot be handled transparently by the platform.sh proxies so you have to take care on your own. If you write an express app, checkout their cors middleware, if you're in PHP nelmio/cors is your best friend but you can also set the headers manually (see `rssreader/index.js`). For the time being it's toally fine to set the `Access-Control-Allow-Origin` header to `*` but that of cors( :P ) must be changed when going live for real. 

#### I'm getting certificate warnings on all https endpoints :(
Have you tried shutting it off and on again? In platform.sh terms that's a `platform redeploy`. By issuing that command new certificate challenges are executed against Lets Encrypt. 

#### Error: watch ... ENOSPC
Uhoh, you ran out of system resources (node_modules, Visual Studio and Docker together can run into this pretty soon). Check this [StackOverflow question](https://stackoverflow.com/questions/16748737/grunt-watch-error-waiting-fatal-error-watch-enospc/17437601#17437601).



