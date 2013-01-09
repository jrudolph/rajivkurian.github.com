---
layout: post
title: "Remote controlled html5 presentation with a Go backend - Part 1."
date: 2013-01-08 22:38
comments: true
categories: [Go, HTML5, websocket]
---
#### What is this?
This is the first part of a three part series on building a remote-controlled presentation app in [Go](http://golang.org). Also see [Part 2](/blog/2013/01/09/remote-controlled-html5-presentation-with-a-go-backend-part-2/) and [Part 3](/blog/2013/01/09/remote-controlled-html5-presentation-with-a-go-backend-part-3/). The source code with instructions on how to run it can be found [here](https://github.com/RajivKurian/remote-presentation).

####Part 1.
At work I am busy writing the connection layer for one of our backend services. We are mainly looking at Scala libraries like [Finagle](http://twitter.github.com/finagle/) and [Spray.io](http://spray.io). I spent some time last year looking at [Go](http://golang.org), especially it's concurrency facilities. Playing with all the Scala networking libraries, made me want to see what Go had to offer. Go comes with a surprisingly complete standard library. Most other things you need are also available as packages. For a thorough introduction to Go's networking libraries please see [Network programming with Go](http://jan.newmarch.name/go/). [Here](http://gary.beagledreams.com/page/go-websocket-chat.html) is a great websocket-chat demo that I looked at for implementation and structure details.
<!-- more -->

I ended up building a application that allows one to remote control a HTML5 presentation. I used the excellent [reveal.js](http://lab.hakim.se/reveal-js) library for the HTML5 presentation part. The reasons for using reveal.js were two fold:

1.  I did not want to write too much HTML to get a complete application working.
2.  The best way to make a trivial demo look great is to pair it with an already impressive library and steal some of it's thunder. Reveal.js does that for my application.

The application is very simple:

1.  A client connects to the server and register with a presentation id. It register's either as a presentation or as a remote-controller.
2.  If we find a corresponding presentation id we add a presentation client's connection to a hub representing that id.
3.  Remote-controller clients can send messages like "left", "right", "up", "down" to navigate all the presentations on a given ub.

Okay let's see some code. We start with our presentation messages. We represent our protocol messages with an int. If we wanted to add more message types this would be the place to start. Our protocol supports basic slide movement messages and a Unknown message for things that are ... well unknown.

``` go presentation.go
package main

import ()

type PresentationMsg int

const (
  Left    = 1
  Right   = 2
  Up      = 3
  Down    = 4
  Unknown = 5
)
```

Lets start with the meat of the application now. We define an interface that represents our connections. These connections could have different implementations. Interfaces in Go tend to be brief. We only have two methods on our interface - one to enable writing messages to a connection and another to clean the connection in case of errors. The connection is responsible for reading messages from the client, deserializing them to protocol messages and sending them to our hub - this is not defined in the interface. We also add some imports that we will  use later.
``` go connection.go
package main

import (
  "code.google.com/p/go.net/websocket"
  "errors"
  "fmt"
  "strings"
)

type Connection interface {
  Write(msg PresentationMsg) error
  Cleanup()
}
```
We start off with a websocket based implementation of the connection interface. The interface is generic enough for us to come up with other implementations like HTTP long polling, HTTP streaming or even TCP. That way browsers which do not support web sockets could still communicate with one's that do. 
``` go connection.go
// Keep pointers to a Hub and a websocket connection.
type WebSocketConnection struct {
  hub *Hub // We will see the Hub implementation later.
  ws  *websocket.Conn
}

// Implements the Cleanup method of the Connection interface.
func (c *WebSocketConnection) Cleanup() {
  c.ws.Close()  // Close the websocket connection.
}

// Implementation of the Write method of the Connection interface.
func (c *WebSocketConnection) Write(msg PresentationMsg) error {
// Serialize our protocol msg.
  var stringMsg string
  switch msg {
  case Left:
    stringMsg = "left"
  case Right:
    stringMsg = "right"
  case Up:
    stringMsg = "up"
  case Down:
    stringMsg = "down"
  // We should only expect to handle slide movement msges here.
  default:
    stringMsg = ""
  }

  if stringMsg != "" {
    err := websocket.Message.Send(c.ws, stringMsg)
    // If there is an error close the connection.
    if err != nil {
      c.Cleanup()
    }
    return err
  }
  return errors.New("Got an unknown msg.")
}
```
Our websocket connection now implements the connection interface and can be treated as one. We still need a way for the websocket connection to read messages from the client and send them to our Hub. We define a method StartReader() to do just this. We specifically try to handle the following messages:

1.  Connection messages used to connect to a particular hub. This can be used by both presentation clients and remote-controller clients.
2.  Remote-control messages used to move the slides. We expect these to only come from remote-controller clients.
``` go connection.go
func (c *WebSocketConnection) StartReader() {
// Keep receiving messages in a for loop unless there is some error.
  for {
    var stringMsg string
    err := websocket.Message.Receive(c.ws, &stringMsg)
    if err != nil {
      break
    }
    fmt.Println("Received a msg " + stringMsg)

    // a "join" msg is of the form "join(Remote/Presentation):presentationId".
    // For eg: joinRemote:123
    if strings.HasPrefix(stringMsg, "joinRemote") {
      presentationId := strings.Split(stringMsg, ":")[1]
      fmt.Println("Msg: JoinRemote: " + presentationId)

      // Again we will see the hub implementation later. We try to retrieve a hub
      // from a global map of hubs otherwise we send an errorMsg to the client and cleanup.
      if hub, ok := hubMap[presentationId]; ok {
        c.hub = &hub  // Keep a pointer to the hub so that it can be used later.
      } else {
        c.writeErrorMsg("Invalid Presentation id: " + presentationId)
        c.Cleanup()
      }
    } else if strings.HasPrefix(stringMsg, "joinPresentation") {
      presentationId := strings.Split(stringMsg, ":")[1]
      fmt.Println("Msg: Join: " + presentationId)
      // If there is an entry for this presentation then register this connection
      // with it otherwise send an error.
      if hub, ok := hubMap[presentationId]; ok {
        c.hub = &hub
        hub.registerConnection(c)
      } else {
        c.writeErrorMsg("Invalid Presentation id: " + presentationId)
        c.Cleanup()
      }
    } else if c.hub != nil {
      // Accept remote msges only if there is a valid presentation hub.
      var msg PresentationMsg
      fmt.Println("Msg: Remote: " + stringMsg)
      switch stringMsg {
      case "left":
        msg = Left
      case "right":
        msg = Right
      case "up":
        msg = Up
      case "down":
        msg = Down
      default:
        msg = Unknown
      }
      if msg != Unknown {
        c.hub.sendMsg(msg)
      } else {
        c.writeErrorMsg("Not a valid Msg: " + stringMsg)
        c.hub.unregisterConnection(c)
        c.Cleanup()
      }
    } else { // Send an error msg and cleanup.
      c.writeErrorMsg("Must register with a valid presentationId before sending msges")
      c.Cleanup()
    }
  }
}
```
Almost done with our websocket connection. We define a function wsHandler that will be later used by our websocket server. The websocket server will create a new go routine with our handler for every websocket connection. For every new websocket connection we create a WebsocketConnection object and call the startReader method on it. Again since our handler is run as a go routine startReader will not block the application. We also make use of go's defer facility to clean up when the handler exits.
``` go connection.go
func wsHandler(ws *websocket.Conn) {
  c := &WebSocketConnection{hub: nil, ws: ws}
  defer func() {
    // Unregister here.
    if c.hub != nil {
      c.hub.unregisterConnection(c)
    }
    fmt.Println("A websocket handler exited")
  }()
  // Start reader.
  c.StartReader()
}
```
We have a working websocket connection now. In Part 2 we'll look at the implementation of the hub that let's remote clients talk to the presentation clients. 