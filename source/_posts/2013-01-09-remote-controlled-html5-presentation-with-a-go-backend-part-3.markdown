---
layout: post
title: "Remote controlled html5 presentation with a Go backend - Part 3."
date: 2013-01-09 01:27
comments: true
categories: [Go, HTML5, websocket]
---

#### What is this?
This is the third part of a three part series on building a remote-controlled presentation app in [Go](http://golang.org). Also see [Part 1](/blog/2013/01/08/remote-controlled-html5-presentation-with-a-go-backend-part-1/) and [Part 2](/blog/2013/01/09/remote-controlled-html5-presentation-with-a-go-backend-part-2/). The source code with instructions on how to run it can be found [here](https://github.com/RajivKurian/remote-presentation).

#### Part 3.
We built a working server in parts 1 and 2. Our server was also capable of serving static files. In this part we will use the excellent [reveal.js](https://github.com/hakimel/reveal.js) library for the actual presentation. I simply cloned https://github.com/hakimel/reveal.js into the directory for my go package and made edits to the index.html file to get a working presentation client.

<!-- more -->
We will use the default presentation that comes with reveal.js and add a layer to control it's behavior. Reveal.js exposes it's API through a Reveal object in JavaScript. We can use this object to move our presentation. It also fires a 'ready' event when the presentation is ready to start navigating. We will listen for this event and in response connect to our server and:

1.  Register as presentation "123". In part 2 this is the only presentation id that we support.
2.  Listen to messages over our websocket connection and react by moving the slides accordingly.

``` html index.html
<!-- a bunch of other stuff that comes in reveal.js here -->
    <script>
      // This is all reveal.js configuration code.
      // Full list of configuration options available here:
      // https://github.com/hakimel/reveal.js#configuration
      Reveal.initialize({
        controls: true,
        progress: true,
        history: true,
        center: true,

        theme: Reveal.getQueryHash().theme, // available themes are in /css/theme
        transition: Reveal.getQueryHash().transition || 'default', // default/cube/page/concave/zoom/linear/none

        // Optional libraries used to extend on reveal.js
        dependencies: [
          { src: 'lib/js/classList.js', condition: function() { return !document.body.classList; } },
          { src: 'plugin/markdown/showdown.js', condition: function() { return !!document.querySelector( '[data-markdown]' ); } },
          { src: 'plugin/markdown/markdown.js', condition: function() { return !!document.querySelector( '[data-markdown]' ); } },
          { src: 'plugin/highlight/highlight.js', async: true, callback: function() { hljs.initHighlightingOnLoad(); } },
          { src: 'plugin/zoom-js/zoom.js', async: true, condition: function() { return !!document.body.classList; } },
          { src: 'plugin/notes/notes.js', async: true, condition: function() { return !!document.body.classList; } }
          // { src: 'plugin/remotes/remotes.js', async: true, condition: function() { return !!document.body.classList; } }
        ]
      });

      // Our stuff starts here.
      var conn;  // Our websocket connection.
      Reveal.addEventListener( 'ready', function(event) {
        if (window["WebSocket"]) {
          conn = new WebSocket("ws://localhost:8080/ws");
          conn.onclose = function(evt) {
            console.log("Connection closed.")
          }
          conn.onopen = function(evt) {
            conn.send("joinPresentation:123")
          }
          conn.onmessage = function(evt) {
            var msg = evt.data
            switch (msg) {
              case "left":
              Reveal.left();
              break;
              case "right":
              Reveal.right();
              break;
              case "up":
              Reveal.up();
              break;
              case "down":
              Reveal.down();
              break;
              default: console.log("Unknown msg " + msg)
            }
          }
        } else {
          console.log("This browser does not support websockets.")
        }
      });
    </script>

  </body>
</html>
```
Next we need to write our remote-controller client. I ripped off the design of the controller on the reveal.js homepage and made a bigger, uglier and less semantic version of it on a file called remote.html. I'll spare you the HTML and CSS horror (see the [source](https://github.com/RajivKurian/remote-presentation/blob/master/remote.html) if you dare). I used this [article](http://css-tricks.com/snippets/css/css-triangle/) to draw CSS triangles. I also copied the color of the controls and the radial gradient from reveal.js. 

{% img /images/remote-control.png %}

I'll focus on the javascript instead. Like before we connect to our server and:

1.  Register as a remote-controller for presentation "123".
2.  Listen to onclick events on our control DOM nodes and send corresponding protocol messages to the server.

Note that we do not need to listen to messages from the server since this is a remote-controller.

``` html remote.html
<script type="text/javascript">
  var conn;
  var isSocketOpen = false;

$(document).ready(function() {
  console.log("document is ready")
  var up = $("#up");
  var down = $("#down");
  var left = $("#left");
  var right = $("#right");
  up.click(function() {
    sendMessage("up");
  });
  down.click(function() {
    sendMessage("down");
  });
  left.click(function() {
    sendMessage("left");
  });
  right.click(function() {
    sendMessage("right");
  });
});

  function sendMessage(direction) {
    console.log("direction is " + direction)
    if (!conn || !isSocketOpen) {
      return;
    }
    conn.send(direction);
  }

$(function() {
  if (window["WebSocket"]) {
    conn = new WebSocket("ws://localhost:8080/ws");
    conn.onclose = function(evt) {
    console.log("Connection lost");
    isSocketOpen = false;
    }
    conn.onopen =function(evt) {
      isSocketOpen = true;
      conn.send("joinRemote:123")
    }
    conn.onmessage = function(evt) {
      console.log("Should not be getting any msges since this is a remote." + evt.data)
    }
  } else {
    console.log("This browser does not support websockets")
  }
});
</script>
```
*Yes I log errors to the console!* We now have all the pieces necessary to run the demo. You can use the remote-controller (possibly on your mobile phone) to navigate the reveal.js presentation in all of it's "transitiony" glory.

This brings the three part series to an end. I thoroughly enjoyed writing this demo and showing it to my friends. This is the first server I wrote in Go and it was a surprisingly pleasant experience. I might write another article about my experience with Go and compare it to Scala. 