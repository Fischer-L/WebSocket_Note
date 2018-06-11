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

  // When a HTTP Upgrade request comes in, delegate it to the `engine.io` server.
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
    
    // `engine.io` delegate the HTTP Upgrade request to ws
    self.ws.handleUpgrade(req, socket, head, function (conn) {
      self.onWebSocket(req, conn);
    });
  };
  ```
  [1] https://github.com/socketio/engine.io/blob/c6247514e231566f70f074b14dccaae4c8aeda13/lib/server.js#L347
  
  [2] https://github.com/websockets/ws/blob/d871bdfdc806122862ee5e2b781989b576771caf/doc/ws.md
  
* In websockets/ws/lib/websocket-server.js [1]
  ```js
  /**
   * Handle a HTTP Upgrade request.
   *
   * @param {http.IncomingMessage} req The request object
   * @param {net.Socket} socket The network socket between the server and client
   * @param {Buffer} head The first packet of the upgraded stream
   * @param {Function} cb Callback
   * @public
   */
  handleUpgrade (req, socket, head, cb) {
    // ... ...
    
    // Check if this is a the HTTP Upgrade request for WebSocket.
    if (
      req.method !== 'GET' || req.headers.upgrade.toLowerCase() !== 'websocket' ||
      !req.headers['sec-websocket-key'] || (version !== 8 && version !== 13) ||
      !this.shouldHandle(req)
    ) {
      return abortHandshake(socket, 400);
    }
    
    // ... ...
    
    // The HTTP Upgrade request for WebSocket has been confirmed,
    // proceed to send the handshake response.
    this.completeUpgrade(extensions, req, socket, head, cb);
  }
  
  /**
   * Upgrade the connection to WebSocket.
   *
   * @param {Object} extensions The accepted extensions
   * @param {http.IncomingMessage} req The request object
   * @param {net.Socket} socket The network socket between the server and client
   * @param {Buffer} head The first packet of the upgraded stream
   * @param {Function} cb Callback
   * @private
   */
  completeUpgrade (extensions, req, socket, head, cb) {
    // ... ...
    
    // Compute the Sec-WebSocket-Accept key for the response 
    // to make sure the server support WebSocket.
    // See [2] for more.
    const key = crypto.createHash('sha1')
      .update(req.headers['sec-websocket-key'] + constants.GUID, 'binary')
      .digest('base64');

    // The handshake response header
    const headers = [
      'HTTP/1.1 101 Switching Protocols',
      'Upgrade: websocket',
      'Connection: Upgrade',
      `Sec-WebSocket-Accept: ${key}`
    ];
    
    // The instance of a WebSocket.
    // See https://github.com/websockets/ws/blob/d871bdfdc806122862ee5e2b781989b576771caf/doc/ws.md#class-websocket
    const ws = new WebSocket(null);
    
    // Does the client ask for subprotocols?
    // See [2] for more.
    var protocol = req.headers['sec-websocket-protocol'];
    if (protocol) {
      // ... ...
      
      if (protocol) {
        headers.push(`Sec-WebSocket-Protocol: ${protocol}`);
        ws.protocol = protocol;
      }
    }
    
    // Are we going to compress data?
    if (extensions[PerMessageDeflate.extensionName]) {
      // ... ...
      headers.push(`Sec-WebSocket-Extensions: ${value}`);
      ws._extensions = extensions;
    }
    
    // ... ...
    
    // Send the handshake response
    socket.write(headers.concat('\r\n').join('\r\n'));
    socket.removeListener('error', socketOnError);
    
    // Build and store this WebSocket instance
    ws.setSocket(socket, head, this.options.maxPayload);
    if (this.clients) {
      this.clients.add(ws);
      ws.on('close', () => this.clients.delete(ws));
    }
    cb(ws);
  }
  ```
  [1] https://github.com/websockets/ws/blob/d871bdfdc806122862ee5e2b781989b576771caf/lib/websocket-server.js
  
  [2] https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API/Writing_WebSocket_servers



