# microservice_go
## Dependencies
Go: 1.8 <br>
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
Visualize and test the json response from each microservices. <br>

front-end/cmd/web/templates/test.page.gohtml (the test script of logger service)
```
  logBtn.addEventListener("click", function() {
      const payload = {
          action: "log",
          log: {
              name: "event",
              data: "Some kind of data",
          }
      }

      const headers = new Headers();
      headers.append("Content-Type", "application/json");

      const body = {
          method: "POST",
          body: JSON.stringify(payload),
          headers: headers,
      }

      fetch("http:\/\/localhost:8080/handle", body)
      .then((response) => response.json())
      .then((data) => {
          sent.innerHTML = JSON.stringify(payload, undefined, 4);
          received.innerHTML = JSON.stringify(data, undefined, 4);
          if (data.error) {
              output.innerHTML += `<br><strong>Error:</strong> ${data.message}`;
          } else {
              output.innerHTML += `<br><strong>Response from broker service</strong>: ${data.message}`;
          }
      })
      .catch((error) => {
          output.innerHTML += "<br><br>Error: " + error;
      })
  })
```
### Broker-Service (ToDo)
### Authentication-Service (ToDo)
### Logger-Service
Connect to MongoDB and communicate with broker-service using some communication protocols such as REST, RPC, and gRPC. <br>
- REST: Use srv.ListenAndServe() and Serve by using logItem function in broker-service/cmd/api/handler.go <br>
- RabbitMQ: Use rabbitmq and Serve by using logEventViaRabbit function in broker-service/cmd/api/handler.go <br>
- RPC: Use rpcListen() and Serve by using logItemViaRPC function in broker-service/cmd/api/handler.go <br>
- gRPC: Use gRPCListen() and Serve by using logItemViaGRPC function in broker-service/cmd/api/handler.go <br>


logger-service/cmd/api/main.go
```
func main() {
	// connect to mongo
	mongoClient, err := connectToMongo()
	if err != nil {
		log.Panic(err)
	}
	client = mongoClient

	// create a context in order to disconnect
	ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
	defer cancel()

	// close connection
	defer func() {
		if err = client.Disconnect(ctx); err != nil {
			panic(err)
		}
	}()

	app := Config{
		Models: data.New(client),
	}

	// Register the RPC Server
	err = rpc.Register(new(RPCServer))
	go app.rpcListen()

	go app.gRPCListen()

	// start web server
	log.Println("Starting service on port", webPort)
	srv := &http.Server{
		Addr:    fmt.Sprintf(":%s", webPort),
		Handler: app.routes(),
	}

	err = srv.ListenAndServe()
	if err != nil {
		log.Panic()
	}

}
```
broker-service/cmd/api/handler.go
```
func (app *Config) logItem(w http.ResponseWriter, entry LogPayload) {
	jsonData, _ := json.MarshalIndent(entry, "", "\t")

	logServiceURL := "http://logger-service/log"

	request, err := http.NewRequest("POST", logServiceURL, bytes.NewBuffer(jsonData))
	if err != nil {
		app.errorJSON(w, err)
		return
	}

	request.Header.Set("Content-Type", "application/json")

	client := &http.Client{}

	response, err := client.Do(request)
	if err != nil {
		app.errorJSON(w, err)
		return
	}
	defer response.Body.Close()

	if response.StatusCode != http.StatusAccepted {
		app.errorJSON(w, err)
		return
	}

	var payload jsonResponse
	payload.Error = false
	payload.Message = "logged"

	app.writeJSON(w, http.StatusAccepted, payload)

}

func (app *Config) logEventViaRabbit(w http.ResponseWriter, l LogPayload) {
	err := app.pushToQueue(l.Name, l.Data)
	log.Println(l.Name, l.Data)
	if err != nil {
		app.errorJSON(w, err)
		return
	}

	var payload jsonResponse
	payload.Error = false
	payload.Message = "logged via RabbitMQ"

	app.writeJSON(w, http.StatusAccepted, payload)
}

func (app *Config) logItemViaRPC(w http.ResponseWriter, l LogPayload) {
	client, err := rpc.Dial("tcp", "logger-service:5001")
	if err != nil {
		app.errorJSON(w, err)
		return
	}

	rpcPayload := RPCPayload{
		Name: l.Name,
		Data: l.Data,
	}

	var result string
	err = client.Call("RPCServer.LogInfo", rpcPayload, &result)
	if err != nil {
		app.errorJSON(w, err)
		return
	}

	payload := jsonResponse{
		Error:   false,
		Message: result,
	}

	app.writeJSON(w, http.StatusAccepted, payload)
}

func (app *Config) LogViaGRPC(w http.ResponseWriter, r *http.Request) {
	var requestPayload RequestPayload

	err := app.readJSON(w, r, &requestPayload)
	if err != nil {
		app.errorJSON(w, err)
		return
	}

	conn, err := grpc.Dial("logger-service:50001", grpc.WithTransportCredentials(insecure.NewCredentials()), grpc.WithBlock())
	if err != nil {
		app.errorJSON(w, err)
		return
	}
	defer conn.Close()

	c := logs.NewLogServiceClient(conn)
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()

	_, err = c.WriteLog(ctx, &logs.LogRequest{
		LogEntry: &logs.Log{
			Name: requestPayload.Log.Name,
			Data: requestPayload.Log.Data,
		},
	})
	if err != nil {
		app.errorJSON(w, err)
		return
	}
	var payload jsonResponse
	payload.Error = false
	payload.Message = "logged"

	app.writeJSON(w, http.StatusAccepted, payload)
}
```

### Mail-Service (ToDo)
### Listener-Service (ToDo)
## Future Works
First, I would like to finish all of the Lecture of "Working with Microservices in Go" and then work on the following.
- [ ] Unit Test of all microservices
- [ ] Comparison of the speed of communication protocal such as REST API, RPC, and gRPC between microservices
