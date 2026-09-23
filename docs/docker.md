# Docker

## Overview

Docker is used to run self-hosted services in isolated containers.

The homelab uses Docker as the main platform for deploying services.

## Current Services

- Uptime Kuma

- Immich

## Concepts Being Learned

- Docker images
- Containers
- Container lifecycle
- Volumes
- Networks
- Port mapping
- Restart policies
- Docker Compose

## Container Management

Common commands:

```bash
docker ps
docker ps -a
docker images
docker logs <container>
docker start <container>
docker stop <container>
docker restart <container>

## Restart Policies

- Uptime Kuma (unless-stoped) 
