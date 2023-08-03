# Running XUI cluster - a simple, high performance and secure live media server in pure Rust.
XIU rtmp-cluster https://github.com/harlanc/xiu.git

We will prepare 3 node cluster to run on k8s.

# Part 1 – create Docker container and run locally with docker-compose.

## Requrements: 
- Install Docker (out of scope of this readme).

Special notes for Kali Linux - https://www.kali.org/docs/containers/installing-docker-on-kali/ 
and https://computingforgeeks.com/install-docker-and-docker-compose-on-kali-linux/ 

## Creating container:

Clone xui to local machine in your working dir:
```
git clone https://github.com/harlanc/xiu.git 
```
- We will use Alpine Rust image from official repo - https://hub.docker.com/_/rust 
- We also need musl-dev and x86_64-unknown-linux-musl - https://levelup.gitconnected.com/create-an-optimized-rust-alpine-docker-image-1940db638a6c 
- For xui we need to add basic toml config file (config_rtmp_httpflv_hls.toml) from examples, enabling all 3 protocols - https://github.com/harlanc/xiu/tree/master#configuration-examples 

## Dockerfile:

```
FROM rust:alpine as builder
COPY ./ ./
WORKDIR /xiu/application/xiu
RUN apk add --no-cache musl-dev openssl-dev && \
    rustup target add x86_64-unknown-linux-musl && \
    cargo build --release --target x86_64-unknown-linux-musl
	
FROM alpine
COPY --from=builder /xiu/target/x86_64-unknown-linux-musl/release/xiu /usr/local/bin/xiu
COPY --from=builder /xiu/application/xiu/src/config/config_rtmp_httpflv_hls.toml /etc/xiu/ config_rtmp_httpflv_hls.toml
CMD ["xiu", "-c", "/etc/xiu/config_rtmp.toml"]
```

Building docker image:
```
docker image build -t xiu:1.0.0 .
```
## Test container build

Run container:
```
docker run -d -p 1935:1935 --name xiu xiu:1.0.0
```
Check:
```
docker ps
docker logs xiu
```

Installing FFmpeg:
```
sudo apt install ffmpeg
```

Downloading sample video:
```
curl -O https://sample-videos.com/video123/mp4/240/big_buck_bunny_240p_30mb.mp4
``` 

According to manual https://github.com/harlanc/xiu/tree/master#push-rtmp we can use use FFmpeg to push a rtmp stream:
Running FFmpeg with video file above:
```
ffmpeg -re -stream_loop -1 -i big_buck_bunny_240p_30mb.mp4 -c:a copy -c:v copy -f flv -flvflags no_duration_filesize rtmp://127.0.0.1:1935/live/test
```

And try to play:
```
ffplay -i rtmp://localhost:1935/live/test
```

## Publish to registry

Login to registry (use our name)
```
docker login –u dmorozov82
```

Calculate timestamp & Dockerfile md5:
```
date --utc -d "2023-07-29 17:30" +%s
1690651800
```

Calculate md5:
```
md5sum Dockerfile
bf8e4c4328664a8ac5a997b0e34b4c1c  Dockerfile
```

Adding Tags and Upload:
```
docker tag xiu:1.0.0 dmorozov82/1690651800-bf8e4c4328664a8ac5a997b0e34b4c1c:1.0.0
docker push dmorozov82/1690651800-bf8e4c4328664a8ac5a997b0e34b4c1c:1.0.0
```

Now our container located at - https://hub.docker.com/r/dmorozov82/1690651800-bf8e4c4328664a8ac5a997b0e34b4c1c 

## Creating cluster with docker-compose

- We will follow this part of manual - https://github.com/harlanc/xiu/tree/master#relay---static-pull
- When we push live stream to node 1, and when we play the stream from node 2 or node 3, it will pull the stream from node 1.

### Cluster Configuration
Firstly create config files for master and slave nodes:
-	Master node – create subfolder “conf/master” and file with name config_rtmp_httpflv_hls.toml (witch we used in dockerfile):

```
# Master Node Config
[rtmp]
enabled = true
port = 1935

# Slaves
[[rtmp.push]]
enabled = true
address = "x-node-2"
port = 1935
[[rtmp.push]]
enabled = true
address = "x-node-3"
port = 1935

# Formats
[hls]
enabled = true
port = 8080

[httpflv]
enabled = true
port = 8081

# Logining
[log]
level = "info"
```

-	Same way for slaves, under “slaves” subfolder: 

```
# SLAVE Node Config
[rtmp]
enabled = true
port = 1935

# Formats
[hls]
enabled = true
port = 8080

[httpflv]
enabled = true
port = 8081

# Logining
[log]
level = "info"
```

### Docker-compose
Create docker-compose environment file “.env”:
```
XIMAGE=dmorozov82/1690651800-bf8e4c4328664a8ac5a997b0e34b4c1c:1.0.0
```
And docker-compose.yml:
```
services:
  x-master:
    container_name: x-master
    image: ${XIMAGE}
    ports:
      - 1935:1935
      - 8081:8081
      - 8080:8080
    volumes:
      - ./conf/master/config_rtmp_httpflv_hls.toml:/etc/xiu/config_rtmp_httpflv_hls.toml

  x-slave-1:
    container_name: x-slave-1
    image: ${XIMAGE}
    ports:
      - 1936:1935
      - 8181:8081
      - 8180:8080
    volumes:
      - ./conf/slave/config_rtmp_httpflv_hls.toml:/etc/slave/config_rtmp_httpflv_hls.toml

  x-slave-2:
    container_name: x-slave-2
    image: ${XIMAGE}
    ports:
      - 1937:1935
      - 8281:8081
      - 8280:8080
    volumes:
      - ./conf/slave/config_rtmp_httpflv_hls.toml:/etc/slave/config_rtmp_httpflv_hls.toml
```

### Run cluster with:
```docker compose up -d```

### Testing Cluster:
Running FFmpeg:
```ffmpeg -re -stream_loop -1 -i big_buck_bunny_240p_30mb.mp4 -c:a copy -c:v copy -f flv -flvflags no_duration_filesize rtmp://127.0.0.1:1935/live/test```


Check rtmp, httpflv & hls from every cluster node:
**master:**
```
ffplay -i rtmp://localhost:1935/live/test
ffplay -i http://localhost:8081/live/test.flv
ffplay -i http://localhost:8080/live/test/test.m3u8
```

**slave-1:**
```
ffplay -i rtmp://localhost:1936/live/test
ffplay -i http://localhost:8181/live/test.flv
ffplay -i http://localhost:8180/live/test/test.m3u8
```

**slave-2**
```
ffplay -i rtmp://localhost:1937/live/test
ffplay -i http://localhost:8281/live/test.flv
ffplay -i http://localhost:8280/live/test/test.m3u8
```

Deployment to AWS EKS will be in a separate repo.