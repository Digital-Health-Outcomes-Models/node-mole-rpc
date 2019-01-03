# mole-rpc
Tiny transport agnostic JSON-RPC 2.0 client and server which can work both in NodeJs, Browser, Electron etc

## Table of contents

- [Features](#features)
- [Basic usage](#basic-usage)
  - [Client (with Proxy support)](#client-with-proxy-support)
  - [Client (without Proxy support)](#client-without-proxy-support)
  - [Server (expose instance)](#server-expose-instance)
  - [Server (expose functions)](#server-expose-functions)
- [Advanced usage](#advanced-usage)
- [Use cases](#use-cases)
- [Usage](#usage)
  - [Client](#client)
     - [Interface description](#client-interface-description)
     - [Browser usage](#clientbrowser)
     - [Notifications](#notifications)
     - [Batches](#batches)
     - [Callback syntactic sugar](#client-callback-syntactic-sugar)
     - [Events](#client-events)
  - [Server](#server)
     - [Interface description](#server-interface-description)
     - [Many interfaces at the same time](#many-interfaces-at-the-same-time)
     - [Using the server as a relay](#using-the-server-as-a-relay)
- [FAQ](#faq)
- [Contributing](#contributing)

## Features
 * Transport agnostic (works with HTTP, MQTT, Websocket, Browser post message etc)
 * Works in NodeJs and in browser
 * You can use it to send request to webworker in your browser
 * Server can use several transports the same time
 * Lightweight
 * Modern API
 * Supports all features of JSON-RPC 2.0 (batches, notifications etc)

## Basic usage

You a lot of working examples you can find here too
https://github.com/koorchik/node-mole-rpc-examples

### Client (with Proxy support)

If you use modern JavaScript you can use proxified client. 
It allows you to do remote calls very similar to local calls 

```javascript
const client = new MoleClient(options);
const greeter = client.proxify();

const pong = await greeter.ping();
const greeting1 = await greeter.hello('John Doe');
const greeting2 = await greeter.asyncHello('John Doe');

// Send JSON RPC notification
await greeter.notify.hello('John Doe');
```

### Client (without Proxy support)

```javascript

const client = new MoleClient(options);

const pong = await client.callMethod('ping');
const greeting1 = await client.callMethod('hello', ['John Doe']);
const greeting2 = await client.callMethod('asyncHello', ['John Doe']);

// Send JSON RPC notification
await client.notify('hello', ['John Doe']);
```

### Server (expose instance)

You can expose instance directly. 
Methods which start with underscore will not be exposed.
Built-in methods of Object base class will not be exposed.  

```javascript
class Greeter {
   ping() {
      return 'pong';
   }

   hello(name) { 
      return `Hi ${name}`; 
   }

   asyncHello(name) {
      return new Promise((resolve, reject) => {
         resolve( this.hello(name) );
      });
   }

   _privateMethod() { 
      // will not be exposed
   }
}

const greeter = new Greeter();

const server = new MoleServer(options);
server.expose(greeter);

await server.run();

```

### Server (expose functions)

You can expose functions directly

```javascript
function ping() {
   return 'pong';
}

function hello(name) { 
   return `Hi ${name}`;
}

function asyncHello(name) {
   return new Promise((resolve, reject) {
      resolve( hello(name) );
   });
}

const server = new MoleServer(options);
server.expose({
   ping,
   hello,
   asyncHello
});

await server.run();
```

## Advanced usage

```javascript

// Conflicting names
// Just the same as "greeter.hello('John Doe')"
// It can be usefull when you have conflicting 
// method names like "notify" or "callMethod"
await greeter.callMethod.hello('John Doe'); 

// PROXY: Run in parallel
const promises = [
   greeter.hello('John Doe');
   greeter.notify.hello('John Doe');
];

const results = await Promise.all(promises);

// NO PROXY: Run in parallel
const promises = [
   client.callMethod('hello', ['John Doe']);
   client.notify('hello', ['John Doe']);
];

const results = await Promise.all(promises);

// RUN BATCH

const results = await client.runBatch([
   ['hello', ['John Doe']],
   ['hello', ['John Doe'], 'notify'],
   ['hello', ['John Doe'], 'callMethod'], // "callMethod" is optional
   [methodName, params, mode]
]);


// Result structure
[
   {success: true, result: 123},
   null, // no response for notification
   {success: false, error: errorObject}
];


// IDEAS FOR FUTURE RELEASES:
// PROXY: Run in batch mode (everything in one JSON RPC Request)
const batch = client.newProxyBatch();

const promises = [
   batch.hello('John Doe');
   batch.hello('John Doe');
]; 

batch.runBatch(); // promises will never be resolved if you will not run batch 

const results = await Promise.all(promises);

// NOT PROXY: Run in batch mode (everything in one JSON RPC Request)
const batch = client.newBatch();

const promises = [
   batch.callMethod('hello', ['John Doe']);
   batch.notify('hello', ['John Doe']);
]; 

batch.runBatch(); // promises will never be resolved if you will not run batch 

const results = await Promise.all(promises);
```

## Use cases

### Case 1: Easy way to communicate with web-workers in your browser

### Case 2: Bypass firewall

### Case 3: Microservices via message broker

### Case 4: Lightweight Inter process communication

### You can use different transport for JSON RPC 

So, your code does not depend on a way you communicate. You can use:
* HTTP
* WebSockets
* MQTT
* TCP
* EventEmitter (communicate within one process in an abstract way)
* Named pipe (FIFOs) 

### You can use several transports the same time.

For exampe, you want an RPC server that accepts connections from your local workers by TCP and from Web browser by websocket. You can pass as many transports as you wish. 

### You can use bidirectional websocket connections.

For example, you want a JSON RPC server which handles remote calls but the same time you want to send commands in opposite direction the same time using the same connection.

So, you can use connection initiated by any of the sides for the server and the client the same time.

### You can easly create own transport.

Transports have simple API as possible, so it is very easy to add a new transport. 
