# WebSocket Note

## Overview
```
+---------+        Http         +-------------+  Use http   +------+
|         |  requests/responses |             |  services   |      |
| Network |<------------------->| http.Server |<------------| User |
|         |          ^          |             |             |      |
+---------+          |          +-------------+             +------+
    ^                |                                         |
    |                | Intercept upgrade requests              |
    |                | for websocket                           |
    |                |          +-------------+  Use websocket | 
    | Websocket      +----------|             |  services      | 
    | connection                |  Socket.IO  |<---------------+
    +-------------------------->|             |
                                +-------------+
                                       |
                   Borrow capabilities |
          to handle websocket protocol |
                                       v
                               +----------------+
                               |                |
                               | webSockets/ws  |
                               |                |
                               +----------------+   
``` 

## How Node socket.io establishes a websocket connection

* Because the establishment of WebSocket relies on HTTP Upgrade request so ...
  * On the server, create a HTTP server, then create a WebSocket server based on that.
  ```js
  var app = require('express')();
  var http = require('http').Server(app);
  var io = require('socket.io')(http);
  
  io.on('connection', function(socket){
    console.log('a user connected');
  });
  
  app.get("/", (req, res) => {
    res.sendFile(PATH_TO_HTML_FILE);
  });

  http.listen(3000, function(){
    console.log('listening on *:3000');
  });
  ```

* In socket.io/lib/index.js,
  * Create the WebSocket server instance
  ```js
  function Server(srv, opts){
    // ... ...
    if (srv) this.attach(srv, opts);
  }
  ```
  
  * Attach/Listen to that incoming HTTP request
  ```js
  Server.prototype.listen = Server.prototype.attach = function(srv, opts) {
    // ... ...
    self.initEngine(srv, opts);
    // ... ...
  }
  
  Server.prototype.initEngine = function(srv, opts){
    // initialize engine (borrow the tranport ability from engine.io
    debug('creating engine.io instance with opts %j', opts);
    this.eio = engine.attach(srv, opts);
    
    // ... ...

    // bind to engine "connection" events
    this.bind(this.eio);
  };
  
  Server.prototype.bind = function(engine){
    this.engine = engine;
    this.engine.on('connection', this.onconnection.bind(this));
    return this;
  };
  ```
  
  * After the connection established(including all jobs in the below steps), create a client represents an incoming transport (engine.io) connection
  ```js
  /**
   * Called with each incoming transport connection.
   *
   * @param {engine.Socket} conn
   * @return {Server} self
   * @api public
   */
  Server.prototype.onconnection = function(conn){
    debug('incoming connection with id %s', conn.id);
    var client = new Client(this, conn);
    client.connect('/');
    return this;
  };
  ```

* engine.io inside socket.io does the jobs of transport-based cross-browser/cross-device bi-directional websocket communications.

  * In engine.io/lib/server.js, intercept the HTTP Upgrade request for WebSocket
  ```js
  /**
   * Captures upgrade requests for a http.Server.
   *
   * @param {http.Server} server
   * @param {Object} options
   * @api public
   */
  Server.prototype.attach = function (server, options) {
    // ... ...
    
    // Remove then cache http request listeners so we can intercept
    // the request for WebSocket later.
    // In NodeJs, http.Server inherits net.Server inherits EventEmitter,
    // The below `server.listeners` comes from EventEmitter.
    var listeners = server.listeners('request').slice(0);
    server.removeAllListeners('request');
    server.on('close', self.close.bind(self));
    server.on('listening', self.init.bind(self));
    
    // Add request handler to intercept http requests
    server.on('request', function (req, res) {
      // ... ...
      
      // Not our business. Pass the http request to cached listeners
      for (var i = 0, l = listeners.length; i < l; i++) {
        listeners[i].call(server, req, res);
      }
    });
    
    // Listen to the HTTP Upgrade request for WebSocket
    if (~self.transports.indexOf('websocket')) {
      server.on('upgrade', function (req, socket, head) {
        if (check(req)) {
          self.handleUpgrade(req, socket, head);
        } else if (false !== options.destroyUpgrade) {
          // default node behavior is to disconnect when no handlers
          // but by adding a handler, we prevent that
          // and if no eio thing handles the upgrade
          // then the socket needs to die!
          setTimeout(function () {
            if (socket.writable && socket.bytesWritten <= 0) {
              return socket.end();
            }
          }, destroyUpgradeTimeout);
        }
      });
    }
  }
  ```
  https://github.com/socketio/engine.io/blob/c6247514e231566f70f074b14dccaae4c8aeda13/lib/server.js#L476
  
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
  
* In websockets/ws/lib/websocket-server.js [1], it performs the real HTTP Upgrade handshake for WebSocket
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
    
    // Build and pass on this WebSocket instance
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


## How Node socket.io sends a websocket message
* On the server when we send a message to the client on the end of the socket.
  ```js
  io.on('connection', function(socket){
    socket.emit("server_msg", { body: "The message body" });
  });
  ```

* In socketio/socket.io/lib/socket.js,
  * Socket.prototype.emit
    ```js
     /**
     * Emits to this client.
     * NOTICE: In fact this `emit` is overriden from the EventEmitter class
     *
     * @return {Socket} self
     * @api public
     */
    Socket.prototype.emit = function(ev){
      // ... ...

      // This is the packet which will be sent to the client.
      var packet = {
        type: (this.flags.binary !== undefined ? this.flags.binary : hasBin(args)) ? parser.BINARY_EVENT : parser.EVENT,
        data: args
      };

      // ... ...

      if (rooms.length || flags.broadcast) {
        // Broadcast the packet to sockets in the same rooms.
        // P.S: "Room" is a concept that groups sockets together in socket.io
        this.adapter.broadcast(packet, {
          except: [this.id],
          rooms: rooms,
          flags: flags
        });
      } else {
        // dispatch packet
        this.packet(packet, flags);
      }
      return this;
    }
    ```

  * Socket.prototype.packet
    ```js
    /**
     * Writes a packet.
     *
     * @param {Object} packet object
     * @param {Object} opts options
     * @api private
     */
    Socket.prototype.packet = function(packet, opts){
      packet.nsp = this.nsp.name;
      opts = opts || {};
      opts.compress = false !== opts.compress;
      // `client` represents the client of this socket associated with
      this.client.packet(packet, opts);
    };
    ```
  
* In socketio/socket.io/lib/client.js, it eventually turns to engine.io to send out the packet
  ```js
    /**
     * Writes a packet to the transport.
     *
     * @param {Object} packet object
     * @param {Object} opts
     * @api private
     */
    Client.prototype.packet = function(packet, opts){
      opts = opts || {};
      var self = this;
      // this writes to the actual connection
      function writeToEngine(encodedPackets) {
        if (opts.volatile && !self.conn.transport.writable) return;
        for (var i = 0; i < encodedPackets.length; i++) {
          // `self.conn` is an instance of Socket in engine.io
          self.conn.write(encodedPackets[i], { compress: opts.compress });
        }
      }

      // ... ...
      
      writeToEngine(packet);
    };
  ```

* In socketio/engine.io/lib/socket.js,
  * `write` actually calls `sendPacket`
    ```js
    Socket.prototype.send =
    Socket.prototype.write = function (data, options, callback) {
      this.sendPacket('message', data, options, callback);
      return this;
    };
    ```
    
  * Flush the packet
    ```js
    Socket.prototype.sendPacket = function (type, data, options, callback) {
      // ... ...

      // Still the socket is open?
      if ('closing' !== this.readyState && 'closed' !== this.readyState) {
        // ... ...

        // Save the packet to the write buffer array.
        // Does this imply the async concept?
        this.writeBuffer.push(packet);

        // ... ...

        this.flush();
      }
    }

    Socket.prototype.flush = function () {
      // Safe to flush?
      if ('closed' !== this.readyState && this.transport.writable && this.writeBuffer.length) {
        // ... ...

        var wbuf = this.writeBuffer;
        this.writeBuffer = [];

        // ... ...

        // `this.transport` is an instance of the subclass of Transport in engien.io,
        // which means the transport method, could be WebSocket or Polling.
        // This is the value of socket.io. It can seamless fallback to Polling if no WebSocket.
        this.transport.send(wbuf);
        this.emit('drain');
        this.server.emit('drain', this);
      }
    };
    ```
 
 * In socketio/engine.io/lib/transports/websocket.js (assume our client supports WebSocket)
   ```js
    WebSocket.prototype.send = function (packets) {
     var self = this;

     for (var i = 0; i < packets.length; i++) {
       var packet = packets[i];
       // Encode packets into the engine.io-defined format then send one by one
       parser.encodePacket(packet, self.supportsBinary, send);
     }

     function send (data) {
       // ... ...

       // NOTICE: Sync or Async?
       // This implies taht must wait for one packet sent then proceed to next one.
       // However if the below `self.socket.send` is async, we might re-enter again
       // before `writable` gets `true` onEnd...
       self.writable = false;
       // `sokcet` is an instance of WebSocket in ws
       self.socket.send(data, opts, onEnd);
     }

     function onEnd (err) {
      if (err) return self.onError('write error', err.stack);
      self.writable = true;
      self.emit('drain');
     }
   };
  
   
* In websockets/ws/lib/websocket.js
  ```js
   send (data, options, cb) {
     // ... ...
    
     // `_sender` is an instance of Sender class in ws
     this._sender.send(data || constants.EMPTY_BUFFER, opts, cb);
   }
  ```

* websockets/ws/lib/sender.js
  * Make sure the data being sent is a nodejs Buffer (binary data)
  ```js
  send (data, options, cb) {
    // ... ...
    
    // Make sure the data being sent is a nodejs Buffer (binary data)
    if (!Buffer.isBuffer(data)) {
      if (data instanceof ArrayBuffer) {
        data = Buffer.from(data);
      } else if (ArrayBuffer.isView(data)) {
        data = viewToBuffer(data);
      } else {
        data = Buffer.from(data);
        readOnly = false;
      }
    }
    
    // ... ...
    
    if (perMessageDeflate) {
      // The case sending compressed data.
      // In fact the WebSocket standard has a standard for this case:
      // https://tools.ietf.org/html/draft-ietf-hybi-permessage-compression-19
      
      // ... ...
    } else {
      this.sendFrame(Sender.frame(data, {
        fin: options.fin,
        rsv1: false,
        opcode,
        mask: options.mask,
        readOnly
      }), cb);
    }
  }
  ```

  * Frame the outgoing data per the HyBi WebSocket protocol. This step is important in that it turns our binary data into a WebSocket frame[1]. See the Frame section on MDN for detail[2].
  ```js
  /**
   * Frames a piece of data according to the HyBi WebSocket protocol.
   *
   * @param {Buffer} data The data to frame
   * @param {Object} options Options object
   * @param {Number} options.opcode The opcode
   * @param {Boolean} options.readOnly Specifies whether `data` can be modified
   * @param {Boolean} options.fin Specifies whether or not to set the FIN bit
   * @param {Boolean} options.mask Specifies whether or not to mask `data`
   * @param {Boolean} options.rsv1 Specifies whether or not to set the RSV1 bit
   * @return {Buffer[]} The framed data as a list of `Buffer` instances
   * @public
   */
  static frame (data, options) {
    const merge = data.length < 1024 || (options.mask && options.readOnly);
    var offset = options.mask ? 6 : 2;
    var payloadLength = data.length;

    if (data.length >= 65536) {
      offset += 8;
      payloadLength = 127;
    } else if (data.length > 125) {
      offset += 2;
      payloadLength = 126;
    }

    const target = Buffer.allocUnsafe(merge ? data.length + offset : offset);

    target[0] = options.fin ? options.opcode | 0x80 : options.opcode;
    if (options.rsv1) target[0] |= 0x40;

    if (payloadLength === 126) {
      target.writeUInt16BE(data.length, 2);
    } else if (payloadLength === 127) {
      target.writeUInt32BE(0, 2);
      target.writeUInt32BE(data.length, 6);
    }

    if (!options.mask) {
      target[1] = payloadLength;
      if (merge) {
        data.copy(target, offset);
        return [target];
      }

      return [target, data];
    }

    const mask = crypto.randomBytes(4);

    target[1] = payloadLength | 0x80;
    target[offset - 4] = mask[0];
    target[offset - 3] = mask[1];
    target[offset - 2] = mask[2];
    target[offset - 1] = mask[3];

    if (merge) {
      bufferUtil.mask(data, mask, target, offset, data.length);
      return [target];
    }

    bufferUtil.mask(data, mask, data, 0, data.length);
    return [target, data];
  }
  ```
  
  [1] WebSocket frame format:
  ```
      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-------+-+-------------+-------------------------------+
     |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
     |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
     |N|V|V|V|       |S|             |   (if payload len==126/127)   |
     | |1|2|3|       |K|             |                               |
     +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
     |     Extended payload length continued, if payload len == 127  |
     + - - - - - - - - - - - - - - - +-------------------------------+
     |                               |Masking-key, if MASK set to 1  |
     +-------------------------------+-------------------------------+
     | Masking-key (continued)       |          Payload Data         |
     +-------------------------------- - - - - - - - - - - - - - - - +
     :                     Payload Data continued ...                :
     + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
     |                     Payload Data continued ...                |
     +---------------------------------------------------------------+
  ```

  [2] https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API/Writing_WebSocket_servers

* Send out the WebSocke frame to the client through a real nodeJs socket
  ```js
  sendFrame (list, cb) {
    // This `_socket` is an instance of the net.Socket in nodeJs, see [1].
    if (list.length === 2) {
      this._socket.write(list[0]);
      this._socket.write(list[1], cb);
    } else {
      this._socket.write(list[0], cb);
    }
  }
  ```
  [1] https://nodejs.org/api/net.html#net_socket_write_data_encoding_callback
  

## How socket.io receives a websocket message
* In websockets/ws/lib/websocket-server.js, when completing the HTTP websoscket upgrade
  ```js
  completeUpgrade (extensions, req, socket, head, cb) {
    // ... ...
    
    const ws = new WebSocket(null);
    
    // ... ...
    
    // Get the nodeJs socket. Do the set up.
    ws.setSocket(socket, head, this.options.maxPayload);
    
    // ... ...
    
    // engine.io will receive this websocket connection.
    cb(ws);
  }
  ```

* In websockets/ws/lib/websocket.js, 
  * Setup to listen to the ndoeJs socket
    ```js
    /**
     * Set up the socket and the internal resources.
     *
     * @param {net.Socket} socket The network socket between the server and client
     * @param {Buffer} head The first packet of the upgraded stream
     * @param {Number} maxPayload The maximum allowed message size
     * @private
     */
    setSocket (socket, head, maxPayload) {
      // ... ...

      // The `receiver` in charge of processing the received websocket data.
      receiver.on('message', receiverOnMessage);

      // ... ...

      // Listen to the ndoeJs socket.
      socket.on('data', socketOnData);

      // ... ...
    }
    ```
  
  * When data come in from a nodeJs socket, call the receiver setup above to process.
    ```js
    function socketOnData (chunk) {
      // The `write` will eventually calls `_write` to "drain" the buffered chunk, see [1].
      if (!this[kWebSocket]._receiver.write(chunk)) {
        this.pause();
      }
    }
    ```

* In websockets/ws/lib/receiver.js
  * Receive the buffered data
    ```js
    _write (chunk, encoding, cb) {
     if (this._opcode === 0x08) return cb();

     this._bufferedBytes += chunk.length;
     this._buffers.push(chunk);
     this.startLoop(cb);
    }
    ```
    [1] https://nodejs.org/api/stream.html#stream_writable_write_chunk_encoding_callback_1
 
   * Start the data-paring loop. This loop is to parse a websocket frame. Basically it is a state machine as below:
     ```js
     //
     //            +------------------------------------------------------------+
     //            |                                                            |
     //            |                    +------------------+------------+       |
     //            V                    |                  |            |       |
     // Initial: GET_INFO --> GET_PAYLOAD_LENGTH_16        |            |       |
     //            |                                       |            |       |
     //            |                                       V            v       |
     //            +--------> GET_PAYLOAD_LENGTH_64 --> GET_MASK --> GET_DATA --+
     //            |                    |                  ^            ^
     //            |                    |                  |            |
     //            +--------------------+------------------+------------+
     //                                 |                  |            |
     //                                 +------------------+------------+
     //
     startLoop (cb) {
       var err;
       this._loop = true;

       do {
         switch (this._state) {
           case GET_INFO:
             err = this.getInfo();
             break;
           case GET_PAYLOAD_LENGTH_16:
             err = this.getPayloadLength16();
             break;
           case GET_PAYLOAD_LENGTH_64:
             err = this.getPayloadLength64();
             break;
           case GET_MASK:
             this.getMask();
             break;
           case GET_DATA:
             err = this.getData(cb);
             break;
           default: // `INFLATING`
             this._loop = false;
             return;
         }
       } while (this._loop);

       cb(err);
     }
     ```

  * Read the data after parsing
    ```js
    getData (cb) {
      var data = constants.EMPTY_BUFFER;

      if (this._payloadLength) {
        // ... ...

        //  Consumes `n` bytes from the buffered data.
        data = this.consume(this._payloadLength);
        if (this._masked) bufferUtil.unmask(data, this._mask);
      }

      // In a websocket frame, the opcdoe defines how to interpret the payload data.
      // From 0x08(a close connection), it means a control frame.
      // See https://tools.ietf.org/html/rfc6455 for details.
      if (this._opcode > 0x07) return this.controlMessage(data);

      if (this._compressed) {
        this._state = INFLATING;
        this.decompress(data, cb);
        return;
      }

      if (data.length) {
        //
        // This message is not compressed so its lenght is the sum of the payload
        // length of all fragments.
        //
        this._messageLength = this._totalPayloadLength;
        this._fragments.push(data);
      }

      return this.dataMessage();
    }
    ```

  * Tell outside the message arrives after reading the data
    ```js
    dataMessage () {
      if (this._fin) { // Are wew finished?
        // ... ...

        // The 0x02 opcode means a binary frame
        if (this._opcode === 2) {
          var data;

          if (this._binaryType === 'nodebuffer') {
            data = toBuffer(fragments, messageLength);
          } else if (this._binaryType === 'arraybuffer') {
            data = toArrayBuffer(toBuffer(fragments, messageLength));
          } else {
            data = fragments;
          }

          this.emit('message', data);
        } else {
          const buf = toBuffer(fragments, messageLength);

          if (!validation.isValidUTF8(buf)) {
            this._loop = false;
            return error(Error, 'invalid UTF-8 sequence', true, 1007);
          }

          this.emit('message', buf.toString());
        }
      }

      this._state = GET_INFO;
    }
    ```

* In websockets/ws/lib/websocket.js, received the "message" event from the receiver, then notify the wewbsocket user that the message comes in
  ```js
  function receiverOnMessage (data) {
    this[kWebSocket].emit('message', data);
  }
  ```
* In socketio/engine.io/lib/transports/websocket.js,
  * Listen to the ws websocket
    ```js
    function WebSocket (req) {
      // ... ...
      this.socket.on('message', this.onData.bind(this));
      // ... ...
    }
    ```

  * Pass on the message packet
    ```js
    WebSocket.prototype.onData = function (data) {
      debug('received "%s"', data);
      Transport.prototype.onData.call(this, data);
    };
    ```

    * In socketio/engine.io/lib/transport.js,
      ```js
      Transport.prototype.onData = function (data) {
        this.onPacket(parser.decodePacket(data));
      };

      Transport.prototype.onPacket = function (packet) {
        this.emit('packet', packet);
      };
      ```

* In socketio/engine.io/lib/socket.js,
  * Listen to the websocket transport,
    ```js
    Socket.prototype.setTransport = function (transport) {
      // ... ...
      this.transport.on('packet', onPacket);
      // ... ...
    }
    ```

  * Pass on the message packet
    ```js
    Socket.prototype.onPacket = function (packet) {
      if ('open' === this.readyState) {
        // export packet event
        debug('packet');
        this.emit('packet', packet);

        // ... ...

        switch (packet.type) {
          case 'ping':
            debug('got ping');
            this.sendPacket('pong');
            this.emit('heartbeat');
            break;

          // ... ...

          case 'message':
            this.emit('data', packet.data);
            this.emit('message', packet.data);
            break;
        }
      } else {
        debug('packet received with closed socket');
      }
    };
    ```

* In socketio/socket.io/lib/client.js,
  * Listen to the engine.io socket
    ```js
    Client.prototype.setup = function(){
      // ... ...

      // The `decoder` is for the socket.io protocol
      this.decoder.on('decoded', this.ondecoded);
      this.conn.on('data', this.ondata);

      // ... ...
    }
    ```

  * Decode the packet data, then pass it on
    ```js
    Client.prototype.ondata = function(data){
      // ... ...
      
      // OK, the decode process is async
      this.decoder.add(data);

      // ... ...
    };

    Client.prototype.ondecoded = function(packet) {
      if (parser.CONNECT == packet.type) {
        this.connect(url.parse(packet.nsp).pathname, url.parse(packet.nsp, true).query);
      } else {
        // ... ...

        process.nextTick(function() {
          socket.onpacket(packet);
        });

        // ... ...
      }
    };
    ```

* socketio/socket.io/lib/socket.js,
  * Handle the packet case by case
    ```js
    Socket.prototype.onpacket = function(packet){
      debug('got packet %j', packet);
      switch (packet.type) {
        case parser.EVENT:
          this.onevent(packet);
          break;

        case parser.BINARY_EVENT:
          this.onevent(packet);
          break;

        case parser.ACK:
          this.onack(packet);
          break;

        case parser.BINARY_ACK:
          this.onack(packet);
          break;

        case parser.DISCONNECT:
          this.ondisconnect();
          break;

        case parser.ERROR:
          this.onerror(new Error(packet.data));
      }
    };
    ```

  * Dispatch incoming event to socket listeners
    ```js
    Socket.prototype.onevent = function(packet){
      var args = packet.data || [];

      // ... ...

      this.dispatch(args);
    };
    
    Socket.prototype.dispatch = function(event){
      // ... ...
      
      // This `emit` is `Emitter.prototype.emit`
      emit.apply(self, event);
      
      // ... ...
    };
    ```
