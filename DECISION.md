# Why and How I did What I Did

## Opening rant

A lot of thought went into this project, and it didn't take quite long to realize that understanding the task itself was the first test. First I had to understand what this blue/green meant from a high level (the one/two slack threads giving real-world metaphors with a kitchen and the other my EC2-instance-fried-brain can't remember).

Running through hours of understanding nginx configurations and engaging in debates with Claude, I was able to finally wrap my head around a *how* for the task.

I ran a simulation myself on my Proxmox homelab setup with an Ubuntu LXC and sample node app.

```js
const express = require('express');
const app = express();

const PORT = process.env.PORT || 3000;
const APP_POOL = process.env.APP_POOL || 'unknown';
const RELEASE_ID = process.env.RELEASE_ID || 'v1.0.0';

let chaosMode = false;
let chaosType = 'error'; // 'error' or 'timeout'

app.get('/version', (req, res) => {
  if (chaosMode) {
    if (chaosType === 'timeout') {
      return;
    }
    return res.status(500).json({ error: 'Service unavailable (chaos mode)' });
  }
  
  res.set('X-App-Pool', APP_POOL);
  res.set('X-Release-Id', RELEASE_ID);
  res.json({
    status: 'ok',
    pool: APP_POOL,
    release: RELEASE_ID,
    timestamp: new Date().toISOString()
  });
});

app.get('/healthz', (req, res) => {
  if (chaosMode) {
    return res.status(500).send('unhealthy');
  }
  res.send('healthy');
});

app.post('/chaos/start', (req, res) => {
  chaosMode = true;
  chaosType = req.query.mode || 'error';
  console.log(`CHAOS MODE ACTIVATED: ${chaosType}`);
  res.json({ 
    chaos: 'started', 
    mode: chaosType,
    pool: APP_POOL 
  });
});

app.post('/chaos/stop', (req, res) => {
  chaosMode = false;
  console.log('Chaos mode stopped');
  res.json({ 
    chaos: 'stopped',
    pool: APP_POOL 
  });
});

app.get('/', (req, res) => {
  res.json({
    service: 'Blue/Green Test App',
    pool: APP_POOL,
    release: RELEASE_ID,
    endpoints: [
      'GET /version',
      'GET /healthz',
      'POST /chaos/start?mode=error|timeout',
      'POST /chaos/stop'
    ]
  });
});

app.listen(PORT, () => {
  console.log(`App running on port ${PORT}`);
  console.log(`Pool: ${APP_POOL}`);
  console.log(`Release: ${RELEASE_ID}`);
});
```

```json
{
  "name": "blue-green-test-app",
  "version": "1.0.0",
  "description": "",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

```Dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package.json .
RUN npm install

COPY app.js .

EXPOSE 3000

CMD ["npm", "start"]
```

Built the image with `docker build -t bluegreen-app:latest` and then created sample `nginx.conf` and `docker-compose.yml` files:

```conf
upstream backend {
    server app_blue:3000 max_fails=2 fail_timeout=5s;
    server app_green:3000 backup max_fails=2 fail_timeout=5s;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://backend;
        proxy_connect_timeout 2s;
        proxy_read_timeout 3s;
        proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
        proxy_next_upstream_tries 2;
        proxy_next_upstream_timeout 10s;
        proxy_pass_request_headers on;
    }
}
```

```yml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - app_blue
      - app_green

  app_blue:
    image: bluegreen-app:latest
    ports:
      - "8081:3000"
    environment:
      - APP_POOL=blue
      - RELEASE_ID=v1.0.0
      - PORT=3000

  app_green:
    image: bluegreen-app:latest
    ports:
      - "8082:3000"
    environment:
      - APP_POOL=green
      - RELEASE_ID=v.1.0.1
      - PORT=3000
```

Then `docker compose up -d` to start the respective blue and green containers.

I tested the endpoints with curl (localhost) and it worked perfectly.

## Stage 2 (Literally)

I read the task one more time to ensure I wasn't missing any tricks or overlooking details, and I was indeed doing just that.

Hiding between the lines, the task required me to use an *existing* docker image (which already has the endpoints baked in), populate nginx's config via environment variables, and ensure a `.env.example` was present in my repo upon submission.

>Docker image? Okay, I get that.
>Nginx config template? Hm. Okay.
>environment variables, yeah sure.
>`ACTIVE_POOL`? Wait, what? How does this work?

Claude highlighted the `envsubst` command, how it fits into all this, and how to use this `ACTIVE_POOL` environment variable to set the active pool (needed for the manual toggle) without overcomplicating the setup. It suggested these two approaches:

- Via manual `envsubst`
- Via Docker compose

I don't think the task requested me to manually run `envsubst`. With more help from Claude, I asked if it was possible to have only an `env` file at root and have everything else work internally and—

>Official nginx image automatically runs envsubst if you use the `.template` suffix! No script needed.
>
>Just name your file `nginx.conf.template` and mount it to `/etc/nginx/templates/`.

```conf
upstream backend_blue_primary {
    server app_blue:3000 max_fails=2 fail_timeout=5s;
    server app_green:3000 backup max_fails=2 fail_timeout=5s;
}

upstream backend_green_primary {
    server app_green:3000 max_fails=2 fail_timeout=5s;
    server app_blue:3000 backup max_fails=2 fail_timeout=5s;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://backend_${ACTIVE_POOL}_primary;
        
        proxy_connect_timeout 2s;
        proxy_read_timeout 3s;
        proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
        proxy_next_upstream_tries 2;
        proxy_next_upstream_timeout 10s;
        proxy_pass_request_headers on;
    }
}
```

I also needed to modify `docker-compose.yml`:

```yml
  volumes:
    - ./nginx.conf.template:/etc/nginx/templates/default.conf.template:ro
  environment:
    - ACTIVE_POOL=${ACTIVE_POOL}
```

Which allows me to switch the active pool by manually editing `.env` or via sed: `sed -i 's/ACTIVE_POOL=.*/ACTIVE_POOL=green/' .env`

```env
BLUE_IMAGE=yimikaade/wonderful:devops-stage-two
GREEN_IMAGE=yimikaade/wonderful:devops-stage-two
ACTIVE_POOL=blue
RELEASE_ID_BLUE=v1.0.0
RELEASE_ID_GREEN=v1.0.1
PORT=3000
```

And the making sure I restart nginx with `docker compose restart nginx`

## Finalizing

I made a copy of `.env` to `.env.example` and added `.env` to `.gitignore` (and moved my previous custom app out the development directory).

Then published to GitHub.

So, all I needed to do in my EC2 instance was:

```bash
git clone https://github.com/me/my-repo.git

cd my-repo

cp .env.example .env

docker compose up -d
```

## Second chance (in retrospect)

Prior to submission, I ran all this on my Proxmox setup and didn't realize the task required me to submit a live IP which meant I had to run all this in my EC2 instance!!!

I did that, submitted... and FAILED!

HOW?

WAIT.

HOWWWW?

Apparently, I had to add those ports to inbound rules due to—

>Oh no! 😭 But this is a CLASSIC cloud deployment mistake - happens to everyone!
>
>**The Problem: AWS Security Groups (Firewall)**
>
>Your app works on localhost inside the EC2, but the grader can't reach it from the internet because:
>
>AWS blocks all ports by default! 🚫
>
>You need to open the ports in your EC2 Security Group.

Honestly, I did not know this either. Failing for this is worth it, but I literally did not realize this (cause I was in a hurry to submit the task at this point).

I didn't understand until submission that this was the reason why the live IP address was requested.

I learnt now that:

>After deploying, always test from outside

I edited the inbound rules by adding ports `8080`, `8081`, and `8082` and setting them to `0.0.0.0/0` to allow traffic from all IPs.

Claude was present to give support:

>You crushed it! The only issue was the security group - a classic AWS gotcha that even senior devs forget sometimes 😅

*Thanks Claude and Thank you @Phoenix and mentors for allowing a resubmission*
