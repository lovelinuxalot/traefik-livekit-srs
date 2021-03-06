# traefik-livekit-srs

This repo contains information how to set this config up and run

## Information

|Service| URL| Purpose| Ports |
|---|---|---|---|
| Traefik |  [https://monitor.flypov.com](https://monitor.flypov.com)| Frontend proxy| `80`,<br> `443`|
| LiveKit Server | [https://lv.flypov.com](https://lv.flypov.com)| Chat app|`7880` (mapped to 443 port via Traefik),<br> `7881`,<br> `7882` (WebRTC)|
| Redis | no url, internal service| LiveKit server room keys management|`6379` (internal port) |
| SRS | [https://srs.flypov.com](https://srs.flypov.com)| Streaming server | `8080` (HTTP streaming port),<br> `1935` (RTMP Streaming Port),<br> `1985` (HTTP API port),<br> `8000/udp` (WebRTC)|
| Postgres | no url, internal service| Hasure GraphQL engine metadata and backend storage | `5432` (internal port)|
| Hasura GraphQL | [https://graphql.flypov.com](https://graphql.flypov.com) | GraphQL engine | `8080` (mapped to 443 port via Traefik) |

<br>

## Procedure

- Clone the repo to a server

<br>

### Setting up for Traefik
- Install the necessary packages
  ```bash
  sudo apt-get install apache2-utils
  ```
- Create a password to secure traefik endpoint. The value of `secure_password` should be replaced by something else
  ```bash
  htpasswd -nb admin secure_password
  ```
- Copy the output and replace it in the file `traefik/traefik_dynamic.toml`

<br>

### Setting up for LiveKit Server

- Generate initial token config by running the command
  ```bash
  docker run --rm -v$PWD:/output livekit/generate --local
  ```

- Compare the config file in `livekit/livekit.yaml` and replace the keys section with appropriate key and the value from the output of the above command

- If the redis password needs to be changed, then please adjust the same password in the docker-compose file in redis section. This is not a problem as redis is not being exposed to the internet and this is a testing app

- A random token will be generated for the room, when running this command, please keep a note of that, as its required for testing.

- Run the command to spin up the containers:
  ```bash
  docker-compose up -d
  ```
- Ensure the containers are running by running
  ```bash
  docker ps 
  ```
- If there are 6 containers running, then you are good to go. The containers would be:
  - traefik
  - livekit
  - redis
  - srs
  - postgres
  - graphql

<br>

## Testing Traefik

- Go to [https://monitor.flypov.com/dashboard/](https://monitor.flypov.com/dashboard/).
- Login credentials as created in previous step with user `admin`
- IF the login is successful and you see a dashboard, then Traefik is up

<br>

## Testing Livekit

- Go to [https://livekit.io/connection-test](https://livekit.io/connection-test).
- Enter the following details
  - LiveKit Url: wss://lv.flypov.com
  - Room Tokem: Paste the random Token generated when setting up LiveKit
  - Click `Start Test`
  - If everything is good, there will be a good response.
  - Please note that during testing you might need to allow audio and video in the browser if they are not allowed by default

<br>

## Testing SRS

- On a local machine, run the command
  ```bash
  docker run --rm ossrs/srs:encoder   ffmpeg -re -i ./doc/source.200kbps.768x320.flv -c copy -f flv rtmp://srs.flypov.com/live/livestream
  ```
- This will download the image and start to run. 
- When its running, open VLC > Go to File > Open Network Stream
- Test the following URLS
  - RTMP: rtmp://rtmp.flypov.com/live/livestream
  - HTTP: http://hls.flypov.com:8080/live/livestream.m3u8

- WebRTC is also enabled on the container, honestly I dont know how to test it, so I have left that to you, if you know how to test it.

<br>

## Testing GraphQL

- Go to [https://graphql.flypov.com](https://graphql.flypov.com)
- Login with the same credentials created for Traefik, because the same middleware that controls the auth for traefik is reused with this service
- Once logged in, the GraphQL console is visible

## Additional information

- There are some improvements mentioned [here](https://docs.livekit.io/deploy/test-monitor#kernel-parameters). This is when running in a production system. Since this is testing, I skipped it

- I have not run livekit and srs at the same time, because this can be CPU intensive when running two servers, So I stopped one and started the other to avoid any performance issues

- Currently HTTPS proxy for SRS does not support with Traefik. This is actually a problem with Traefik, where regex is not optimal fore replace or redirect of urls. SRS recommeneded HTTPS proxy is NGINX. So for now only http urls work for SRS

- I took the liberty to connect the postgres database for the GraphQL service. The Database URL to connect is found in the graphql service section in Docker compose file.
