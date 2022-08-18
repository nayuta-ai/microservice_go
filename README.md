# microservice_go
## Dependencies
Go: 1.8
Docker: 20.0

## Steps
1. install
```
git clone https://github.com/nayuta-ai/microservice_go.git
```
2. Environment Setup
```
cd project
make up_build
```
3. Start Front-end
```
make start
```
Access http://localhost from your browser.

## Components
### Front-End
### Broker-Service
### Authentication-Service
### Logger-Service
### Mail-Service
### Listener-Service
## Future Works
First, I would like to finish all of the Lecture of "Working with Microservices in Go" and then work on the following.
- [ ] Unit Test of all microservices
- [ ] Comparison of the speed of communication protocal such as REST API, RPC, and gRPC between microservices
