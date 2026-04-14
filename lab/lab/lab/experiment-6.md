# Experiment 6 – Docker Networking

## Objective
To understand Docker networking.

## Steps

### Create network
docker network create my-network

### Run container 1
docker run -dit --name c1 --network my-network nginx

### Run container 2
docker run -dit --name c2 --network my-network ubuntu

### Test connection
docker exec -it c2 bash
ping c1

## Conclusion
Containers can communicate using Docker network.
