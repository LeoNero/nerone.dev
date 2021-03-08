---
title: "Creating a JSON-RPC 2.0 server in Go"
date: 2021-02-20T01:32:07-03:00
tags: []
draft: false
---

I recently had to create a JSON-RPC 2.0 server using Golang. However, I didn't find much information online, so I decided to document what I was able to come up with :)

You can see the specification of JSON-RPC 2.0 here https://www.jsonrpc.org/specification

### What we are building
We are going to create a simple TO-DO manager (I know, everybody does it :P). It is going to have two simple procedures: `TodoService.AddItem`, and `TodoService.GetItem`. Using [httpie](https://httpie.io/), the requests look like this:
```
http POST http://localhost:8080/jsonrpc jsonrpc="2.0" method="TodoService.AddItem" params:='{"description": "Sleep"}' id=2
http POST http://localhost:8080/jsonrpc jsonrpc="2.0" method="TodoService.GetItem" params:='{"todo_id": 1}' id=2
```

The first request returns:
```json
{
    "id": "2",
    "jsonrpc": "2.0",
    "result": {
        "message": "Item was successfully added",
        "todo_id": 1
    }
}
```

and the second one returns:
```json
{
    "id": "2",
    "jsonrpc": "2.0",
    "result": {
        "description": "Sleep"
    }
}
```

If you provide an invalid `todo_id`, you will get the following response:
```json
{
    "error": {
        "code": -32000,
        "data": null,
        "message": "Could not find item!"
    },
    "id": "2",
    "jsonrpc": "2.0"
}
```

*PS: the `id` parameter in the body of the POST has **nothing** to do with the id of the `todo`; the parameter is necessary in order to receive a response from the server; if you don't provide it, it means you are sending a `notification`, which means you don't care about the server's response (see the JSON-RPC 2.0 documentation for more).*

### Code
First, run `go mod init you-choose-the-name` inside a folder. Then, add the following `main.go` file:
```golang
package main

import (
	"errors"
	"flag"
	"log"
	"net/http"

	"github.com/gorilla/rpc/v2"
	"github.com/gorilla/rpc/v2/json2"
)

type Todo struct {
	Description string `json:"description"`
}

type TodoService struct {
	todos     map[int]*Todo
	todoCount int
}

type AddTodoReply struct {
	TodoID  int    `json:"todo_id"`
	Message string `json:"message"`
}

type GetTodoReq struct {
	TodoID int `json:"todo_id"`
}

func (tm *TodoService) AddItem(r *http.Request, req *Todo, reply *AddTodoReply) error {
	itemId := tm.todoCount + 1
	tm.todoCount += 1

	tm.todos[itemId] = req

	reply.TodoID = itemId
	reply.Message = "Item was successfully added"

	return nil
}

func (tm *TodoService) GetItem(r *http.Request, req *GetTodoReq, reply *Todo) error {
	if todo, found := tm.todos[req.TodoID]; found {
		*reply = *todo
		return nil
	}

	return errors.New("Could not find item!")
}

func main() {
	address := flag.String("address", ":8080", "")

	s := rpc.NewServer()

	todoService := &TodoService{
		todos: make(map[int]*Todo),
	}

	s.RegisterCodec(json2.NewCodec(), "application/json")
	s.RegisterService(todoService, "")

	http.Handle("/jsonrpc", s)

	log.Fatal(http.ListenAndServe(*address, nil))
}
```

Run `go run main.go` and interact with it.
