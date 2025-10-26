# HNG13 Stage 2 DevOps - Blue/Green with Nginx Upstreams (Auto-Failover + Manual Toggle)

## Setup and run

- Ensure dependencies are installed on the remote machine (git, docker, docker-compose and nginx)
- Clone this repo (`git clone https://github.com/Clue-ess-coder/hng13-stage2-devops`)
- Rename `.env.example` to `.env` (and change variables if desired e.g. `ACTIVE_POOL`)
- Change directory into cloned repo and run `docker compose up`

## Verification

Run the following commands to test proper workings. (Replace `<PUBLIC_IP_ADDRESS>` with your instance's public IP address or `localhost` if testing locally.)

```sh
curl -s http://<PUBLIC_IP_ADDRESS>:8080/version                         # responds with green

curl -X POST http://<PUBLIC_IP_ADDRESS>:8081/chaos/start?mode=error     # or mode=timeout

curl -s http://<PUBLIC_IP_ADDRESS>:8080/version                         # should change to green

curl -X POST http://<PUBLIC_IP_ADDRESS>:8081/chaos/stop                 # successfully stops chaos on blue

curl -s http://<PUBLIC_IP_ADDRESS>:8080/version                         # should change back to blue
```

## DISCLAIMER

If using an Amazon EC2 instance with a security group, make sure to add ports `8080`, `8081`, and `8082` to the **inbound rules** table (else requests via the public IP **will not** work—don't ask how I know that).
