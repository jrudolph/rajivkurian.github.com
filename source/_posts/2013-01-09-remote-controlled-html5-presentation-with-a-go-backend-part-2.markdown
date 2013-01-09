---
layout: post
title: "Remote controlled html5 presentation with a Go backend - Part 2."
date: 2013-01-09 00:17
comments: true
categories: [Go, HTML5, websocket]
---
#### What is this?
This is the second part of a three part series on building a remote-controlled presentation app in [Go](http://golang.org). Also see [Part 1](/blog/2013/01/08/remote-controlled-html5-presentation-with-a-go-backend-part-1/) and [Part 3](/blog/2013/01/09/remote-controlled-html5-presentation-with-a-go-backend-part-3/). The source code with instructions on how to run it can be found [here](https://github.com/RajivKurian/remote-presentation).

#### Part 2.
In Part 1 we implemented a websocket connection that would let a client send and receive our protocol messages. We saw sporadic references to a hub that was supposed to manage the communication between remote-controller clients and presentation clients. This hub needs to:

1.  Have a way for presentation connections to register or unregister themselves.
2.  Have a way to broadcast messages from remote-controller clients to presentation clients.

<!-- more -->
Let's look at the implementation. As always we start off with our imports. A hub contains a map with a Connection key (covered in [Part 1](/blog/2013/01/08/remote-controlled-html5-presentation-with-a-go-backend-part-1/)) and a boolean value. Go does not have a set data structure in the standard library. Using a map seems to be a standard way of getting it to work like a set. We use this map/set to keep a track of the hub's connections. We have three channels that are used to send the hub messages. We define register/unregister channels that are used to add/remove connections from the hub. We also have a remote channel that can be used by a remote-controller client to broadcast messages to the presentation clients on this hub. Note: I use "client" and "connection" interchangeably since each connection actually represents a client.
``` go hub.go
package main

import (
  "code.google.com/p/go.net/websocket"
  "flag"
  "fmt"
  "net/http"
)

type Hub struct {
  presentations map[Connection]bool
  register      chan Connection
  unregister    chan Connection
  remote        chan PresentationMsg
}
```
Let's add some methods to our Hub so that our connections can call these methods to interact with a hub instead of sending it messages on it's channels. This will let us transparently change the implementation later.
``` go hub.go
func (h *Hub) registerConnection(c Connection) {
  h.register <- c
}

func (h *Hub) unregisterConnection(c Connection) {
  h.unregister <- c
}

func (h *Hub) sendMsg(msg PresentationMsg) {
  h.remote <- msg
}
```
Now that we have a way to receive and handle messages. We define a run method that will accomplish this. Later when using a hub we need to make sure that each active hub's run method is running as a go routine so that it accepts and processes messages sent to it. Register messages add a connection to the hub and unregister messages remove them. Remote messages come from a remote-controller client and are broadcast to presentation clients. Note that multiple remote-controller clients could potentially connect to the same hub and send messages messing things up. We could ensure that only one remote-control client connects to a hub but for the purpose of the demo we will live with this limitation.
``` go hub.go
func (h *Hub) run() {
  for {
    select {
    case c := <-h.register:
      h.presentations[c] = true
    case c := <-h.unregister:
      delete(h.presentations, c)
    case msg := <-h.remote:
      for p := range h.presentations {
        err := p.Write(msg) // The connection will clean itself if there is a write error.
        if err != nil {
          delete(h.presentations, p)
        }
      }
    }
  }
}
```
Lets create a global map of hubs where hubs are stored by their presentation ids. Let's also create a single hub for our demo.
``` go hub.go
var hubMap = make(map[string]Hub)

var h = Hub{
  presentations: make(map[Connection]bool),
  register:      make(chan Connection),
  unregister:    make(chan Connection),
  remote:        make(chan PresentationMsg),
}
```
Let's add our hub to the hub map with a presentation id of "123". We need to run go routines for each of our hubs (only 1 for this demo) to make sure they listen to messages sent to them. Finally we set up a HTTP and websocket server on port 8080. The HTTP server also serves all the files required for our web clients.
``` go hub.go
var addr = flag.String("addr", ":8080", "http service address")

func main() {
  flag.Parse()

  // For this demo we have a single id that presentations and remotes can connect to.
  hubMap["123"] = h

  for hubKey := range hubMap {
    hubValue := hubMap[hubKey]
    fmt.Println("Running hub with id: " + hubKey)
    go hubValue.run()
  }
  http.Handle("/", http.FileServer(http.Dir("./")))
  http.Handle("/ws", websocket.Handler(wsHandler))
  http.ListenAndServe(":8080", nil)
}
```
This concludes Part 2. We have a running server that can connect presentations to their remote-controllers. In Part 3 we will write some Javascript to connect to our server. We will listen to messages from the server and move our presentation slides using the [reveal.js](http://lab.hakim.se/reveal-js) API.