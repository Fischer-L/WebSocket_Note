# WebSocket Note

## How Node socket.io establishes a websocket connect

* engine.io inside socket.io does the jobs of transport-based cross-browser/cross-device bi-directional websocket communications.

  * On the server
  ```js
  var engine = require('engine.io');
  var server = new engine.Server();

  server.on('connection', function(socket){
    socket.send('hi');
  });

  // When a HTTP Upgrade request for WebSocket comes in,
  // delegate to the `engine.io` server.
  httpServer.on('upgrade', function(req, socket, head){
    server.handleUpgrade(req, socket, head);
  });
  httpServer.on('request', function(req, res){
    server.handleRequest(req, res);
  });
  ```
  
  * In engine.io/lib/server.js [1] : It borrows from WebSocket[2] project.
  
  ```js
  Server.prototype.handleUpgrade = function (req, socket, upgradeHead) {
    // ... ...
    
    // delegate to ws
    self.ws.handleUpgrade(req, socket, head, function (conn) {
      self.onWebSocket(req, conn);
    });
  };
  ```
  [1] https://github.com/socketio/engine.io/blob/c6247514e231566f70f074b14dccaae4c8aeda13/lib/server.js#L347
  
  [2] https://github.com/websockets/ws/blob/d871bdfdc806122862ee5e2b781989b576771caf/doc/ws.md
  



