![nact Logo](https://raw.githubusercontent.com/ncthbrt/nact/master/assets/logo.svg?sanitize=true)

**let reason-nact = (node.js, reason, actors) ⇒ your µ services have never been so typed**\
*your services have never been so µ*

> Please Note: This wrapper is very alpha level. It is highly subject to change and is not yet recommended for any serious use.

<!-- Badges -->

> Note:
>
> Any and all feedback, comments and suggestions are welcome. Please open an issue if you
> find anything unclear or misleading in the documentation. 

# Sponsored by 
[![Root Logo](https://raw.githubusercontent.com/ncthbrt/nact/master/root-logo.svg?sanitize=true)](https://root.co.za)

# Table of Contents
  * [Introduction](#introduction)
  * [Core Concepts](#core-concepts)
    * [Getting Started](#getting-started)
    * [Stateful Actors](#stateful-actors)
    * [Actor Communication](#actor-communication)
    * [Querying](#querying)
    * [Hierarchy](#hierarchy)
    * [Persistence](#persistence)
  * Patterns and Practises
  * [API](#api)
    * [Functions](#functions)
    * [References](#references)
    * [Internal Context](#internal-context)

# Introduction

Nact is an implementation of the actor model for Node.js. It is inspired by the approaches taken by [Akka](getakka.net) and [Erlang](https://erlang.com). Additionally it attempts to provide a familiar interface to users coming from Redux. This project provides a wrapper around nact for those using Bucklescript and/or ReasonML.

The goal of the project is to provide a simple way to create and reason about µ-services and asynchronous event driven architectures in Node.js.

The actor model describes a system made up of a set of entities called actors. An actor could be described as an independently running packet of state. Actors communicate solely by passing messages to one another.  Actors can also create more (child) actors.

Actor systems have been used to drive hugely scalable and highly available systems (such as WhatsApp and Twitter), but that doesn't mean it is exclusively for companies with big problems and even bigger pockets. Architecting a system using actors should be an option for any developer considering considering a move to a µ-services architecture:

  * Creating a new type of actor is a very lightweight operation in contrast to creating a whole new microservice.
  * [Location transparency](https://doc.akka.io/docs/akka/2.5.4/java/general/remoting.html) and no shared state mean that it is possible to defer decisions around where to deploy a subsystem, avoiding the commonly cited problem of prematurely choosing a [bounded context](https://vimeo.com/74589816).
  * Using actors mean that the spaghetti you might see in a monolithic system is far less likely to happen in the first place as message passing creates less coupled systems. 
  * Actors are asynchronous by design and closely adhere to the principles enumerated in the [reactive manifesto](https://www.reactivemanifesto.org/)
  * Actors deal well with both stateful and statelessness, so creating a smart cache, an in-memory event store or a stateful worker is just as easy as creating a stateless db repository layer without increasing infrastructural complexity.

## Caveats

While network transparency and clustering are planned features of the framework, they have not been implemented yet.

# Core Concepts

## Getting Started
[![Remix on Glitch](https://cdn.glitch.com/2703baf2-b643-4da7-ab91-7ee2a2d00b5b%2Fremix-button.svg)](https://glitch.com/edit/#!/remix/nact-stateless-greeter)

> Tip: The remix buttons like the one above, allow you to try out the samples in this guide and make changes to them. 
> Playing around with the code is probably the best way to get to grips with the framework. 

Nact has only been tested to work on Node 8 and above. You can install nact in your project by invoking the following:

```bash
npm install --save reason-nact
```

Once installed, you need to import the start function, which starts and then returns the actor system.

```ocaml
open Nact;
let system = start();
```

Once you have a reference to the system, it is now possible to create our first actor. To create an actor you have to `spawn` it.  As is traditional, let us create an actor which says hello when a message is sent to it. Since this actor doesn't require any state, we can use the simpler `spawnStateless` function.

```ocaml
type greetingMsg = { name: string };

let greeter: actorRef(_, unit) = spawnStateless(
  ~name = "greeter",
  system,
  ({ name }, _) => print_endline("Hello " ++ name)
);
```

The first unamed argument to `spawnStateless` is the parent, which is in this case the actor system. The [hierarchy](#hierarchy) section will go into more detail about this.

The second unamed argument to `spawnStateless` is a function which is invoked when a message is received.

The name argument to `spawnStateless` is optional, and if omitted, the actor is automatically assigned a name by the system.

To communicate with the greeter, we need to `dispatch` a message to it informing it who we are:

```ocaml
dispatch(greeter, { name: "Erlich Bachman" });
```

This should print `Hello Erlich Bachman` to the console. 

To complete this example, we need to shutdown our system. We can do this by calling `stop(system)`

> Note: Stateless actors can service multiple requests at the same time. Statelessness means that such actors do not have to cater for concurrency issues.

## Stateful Actors
[![Remix on Glitch](https://cdn.glitch.com/2703baf2-b643-4da7-ab91-7ee2a2d00b5b%2Fremix-button.svg)](https://glitch.com/edit/#!/remix/nact-stateful-greeter)

One of the major advantages of an actor system is that it offers a safe way of creating stateful services. A stateful actor is created using the `spawn` function.

In this example, the state is initialized to an empty object. Each time a message is received by the actor, the current state is passed in as the first argument to the actor.  Whenever the actor encounters a name it hasn't encountered yet, it returns a copy of previous state with the name added. If it has already encountered the name it simply returns the unchanged current state. The return value is used as the next state.

```ocaml
let statefulGreeter: actorRef(_, unit) =
  spawn(
    ~name="stateful-greeter",
    system,
    (state, {name}, ctx) => {
      let hasPreviouslyGreetedMe = List.exists((v) => v === name, state);
      if (hasPreviouslyGreetedMe) {
        Js.log("Hello again " ++ name);
        state
      } else {
        Js.log("Good to meet you, " ++ name ++ ". I am the " ++ ctx.name ++ " service!");
        [name, ...state]
      }
    },
    []
  );
```

If no state is returned or the state returned is `undefined` or `null`, stateful actors automatically shut down.

Another feature of stateful actors is that you can subscribe to state changes by using the `state$` function. `state$(actor)` returns a [RxJS](http://reactivex.io/rxjs/manual/index.html) observable stream, which makes it very composable. You can map, filter, combine, throttle and perform many other operations on the stream. For example, you could create a subscription to the statefulGreeter which prints a count of the number of unique names which have been greeted:

```js
state$(statefulGreeter)
               .map(state => Object.keys(state).length)
               .subscribe(count => console.log(`The statefulGreeter has now greeted ${count} unique names`));
```

## Actor Communication
[![Remix on Glitch](https://cdn.glitch.com/2703baf2-b643-4da7-ab91-7ee2a2d00b5b%2Fremix-button.svg)](https://glitch.com/edit/#!/remix/nact-ping-pong)

An actor alone is a somewhat useless construct; actors need to work together. Actors can send messages to one another by using the `dispatch` method. 

The third parameter of `dispatch` is the sender. This parameter is very useful in allowing an actor to service requests without knowing explicitly who the sender is.

In this example, the actors Ping and Pong are playing a perfect ping-pong match. To start the match, we dispatch a message to Ping as Pong using the sender parameter. 


```ocaml
let ping =
  spawnStateless(
    ~name="ping",
    system,
    (msg, ctx) => {
      print_endline(msg);
      optionallyDispatch(~sender=ctx.self, ctx.sender, ctx.name)
    }
  );

let pong =
  spawnStateless(
    ~name="pong",
    system,
    (msg, ctx) => {
      print_endline(msg);
      optionallyDispatch(~sender=ctx.self, ctx.sender, ctx.name)
    }
  );

dispatch(~sender=pong, ping, "hello");

Js.Global.setTimeout(() => stop(system), 100);
```
This produces the following console output:

``` 
begin
ping
pong
ping
pong
ping
etc...
```

## Querying

[![Remix on Glitch](https://cdn.glitch.com/2703baf2-b643-4da7-ab91-7ee2a2d00b5b%2Fremix-button.svg)](https://glitch.com/edit/#!/remix/nact-contacts-1)

Actor systems don't live in a vacuum, they need to be available to the outside world. Commonly actor systems are fronted by REST APIs or RPC frameworks. REST and RPC style access patterns are blocking: a request comes in, it is processed, and finally returned to the sender using the original connection. To help bridge nact's non blocking nature, references to actors have a `query` function. Query returns a promise.

Similar to `dispatch`, `query` pushes a message on to an actor's mailbox, but differs in that it also creates a virtual actor. When this virtual actor receives a message, the promise returned by the query resolves. 

In addition to the message, `query` also takes in a timeout value measured in milliseconds. If a query takes longer than this time to resolve, it times out and the promise is rejected. A time bounded query is very important in a production system; it ensures that a failing subsystem does not cause cascading faults as queries queue up and stress available system resources.

In this example, we'll create a simple single user in-memory address book system. To make it more realistic, we'll host it as an express app. You'll need to install `express`, `body-parser`, `uuid` and of course `nact` using npm to get going.

> Note: We'll expand on this example in later sections.

What are the basic requirements of a basic address book API? It should be able to:
 - Create a new contact 
 - Fetch all contacts
 - Fetch a specific contact
 - Update an existing contact
 - Delete a contact

To implement this functionality, we'll need to create the following endpoints:
  - POST `/api/contacts` - Create a new contact 
  - GET `/api/contacts` - Fetch all contacts
  - GET `/api/contacts` - Fetch a specific contact
  - PATCH `/api/contacts/:contact_id` - Update an existing contact
  - DELETE `/api/contacts/:contact_id` - Delete a contact

Here are the stubs for setting up the server and endpoints:

```js
const express = require('express');
const app = express();
const bodyParser = require('body-parser');

app.use(bodyParser.json());

app.get('/api/contacts', (req,res) => { /* Fetch all contacts */ });

app.get('/api/contacts/:contact_id', (req,res) => { /* Fetch specific contact */ });

app.post('/api/contacts', (req,res) => { /* Create new contact */ });

app.patch('/api/contacts/:contact_id',(req,res) => { /* Update existing contact */ });

app.delete('api/contacts/:contact_id', (req,res) => { /* Delete contact */ });

app.listen(process.env.PORT || 3000, function () {
  console.log(`Address book listening on port ${process.env.PORT || 3000}!`);
});
```

Because actor are message driven, let us define the message types used between the api and actor system:

```ocaml
type contactMsg =
  | CreateContact(contact)
  | RemoveContact(contactId)
  | UpdateContact(contactId, contact)
  | FindContact(contactId);

type contactResponseMsg =
  | Success(contact)
  | NotFound;
```
Our contacts actor will need to handle each message type:

```ocaml
type contactsServiceState = {
  contacts: ContactIdMap.t(contact),
  seqNumber: int
};

let createContact = ({contacts, seqNumber}, contact, ctx: ctx(_, _, _, _, _)) => {
  let contactId = ContactId(seqNumber);
  optionallyDispatch(ctx.sender, (contactId, Success(contact)));
  let nextContacts = ContactIdMap.add(contactId, contact, contacts);
  {contacts: nextContacts, seqNumber: seqNumber + 1}
};

let removeContact = ({contacts, seqNumber}, contactId, ctx: ctx(_, _, _, _, _)) => {
  let nextContacts = ContactIdMap.remove(contactId, contacts);
  let msg =
    if (contacts === nextContacts) {
      let contact = ContactIdMap.find(contactId, contacts);
      (contactId, Success(contact))
    } else {
      (contactId, NotFound)
    };
  optionallyDispatch(ctx.sender, msg);
  {contacts: nextContacts, seqNumber}
};

let updateContact = ({contacts, seqNumber}, contactId, contact, ctx: ctx(_, _, _, _, _)) => {
  let nextContacts =
    ContactIdMap.remove(contactId, contacts) |> ContactIdMap.add(contactId, contact);
  let msg =
    if (nextContacts === contacts) {
      (contactId, Success(contact))
    } else {
      (contactId, NotFound)
    };
  optionallyDispatch(ctx.sender, msg);
  {contacts: nextContacts, seqNumber}
};

let findContact = ({contacts, seqNumber}, contactId, ctx: ctx(_, _, _, _, _)) => {
  let msg =
    try (contactId, Success(ContactIdMap.find(contactId, contacts))) {
    | Not_found => (contactId, NotFound)
    };
  optionallyDispatch(ctx.sender, msg);
  {contacts, seqNumber}
};

let system = start();

let contactsService =
  spawn(
    ~name="contacts",
    system,
    (state, msg, ctx) =>
      switch msg {
      | CreateContact(contact) => createContact(state, contact, ctx)
      | RemoveContact(contactId) => removeContact(state, contactId, ctx)
      | UpdateContact(contactId, contact) => updateContact(state, contactId, contact, ctx)
      | FindContact(contactId) => findContact(state, contactId, ctx)
      },
    {contacts: ContactIdMap.empty, seqNumber: 0}
  );

```

Now to wire up the contact service to the API controllers, we have create a query for each endpoint. For example here is how to wire up the fetch a specific contact endpoint (the others are very similar):

```js
app.get('/api/contacts/:contact_id', async (req,res) => { 
  const contactId = req.params.contact_id;
  const msg = { type: GET_CONTACT, contactId };
  try {
    const result = await query(contactService, msg, 250); // Set a 250ms timeout
    switch(result.type) {
      case SUCCESS: res.json(result.payload); break;
      case NOT_FOUND: res.sendStatus(404); break;
      default:
        // This shouldn't ever happen, but means that something is really wrong in the application
        console.error(JSON.stringify(result));
        res.sendStatus(500);
        break;
    }
  } catch (e) {
    // 504 is the gateway timeout response code. Nact only throws on queries to a valid actor reference if the timeout 
    // expires.
    res.sendStatus(504);
  }
});
```
Now this is a bit of boilerplate for each endpoint, but could be refactored so as to extract the error handling into a separate function (called `performQuery`, for the full definition, click on the glitch button). This would allow us to define the endpoints as follows:

```js

app.get('/api/contacts', (req,res) => performQuery({ type: GET_CONTACTS }, res));

app.get('/api/contacts/:contact_id', (req,res) => 
  performQuery({ type: GET_CONTACT, contactId: req.params.contact_id }, res)
);

app.post('/api/contacts', (req,res) => performQuery({ type: CREATE_CONTACT, payload: req.body }, res));

app.patch('/api/contacts/:contact_id', (req,res) => 
  performQuery({ type: UPDATE_CONTACT, contactId: req.params.contact_id, payload: req.body }, res)
);

app.delete('/api/contacts/:contact_id', (req,res) => 
  performQuery({ type: REMOVE_CONTACT, contactId: req.params.contact_id }, res)
);
```

This should leave you with a working but very basic contacts service. 

## Hierarchy

[![Remix on Glitch](https://cdn.glitch.com/2703baf2-b643-4da7-ab91-7ee2a2d00b5b%2Fremix-button.svg)](https://glitch.com/edit/#!/remix/nact-contacts-2)

The application we made in the [querying](#querying) section isn't very useful. For one it only supports a single user's contacts, and secondly it forgets all the user's contacts whenever the system restarts. In this section we'll solve the multi-user problem by exploiting an important feature of any blue-blooded actor system: the hierachy.

Actors are arranged hierarchially, they can create child actors of their own, and accordingly every actor has a parent. The lifecycle of an actor is tied to its parent; if an actor stops, then it's children do too.

Up till now we've been creating actors which are children of the actor system (which is a pseudo actor). However in a real system, this would be considered an anti pattern, for much the same reasons as placing all your code in a single file is an anti-pattern. By exploiting the actor hierarchy, you can enforce a separation of concerns and encapsulate system functionality, while providing a coherent means of reasoning with failure and system shutdown. 

Let us imagine that the single user contacts service was simple a part of some larger system; an email campaign management API for example.  A potentially valid system could perhaps be represented by the diagram below. 

<img height="500px" alt="Example of an Actor System Hierarchy" src="https://raw.githubusercontent.com/ncthbrt/nact/master/assets/hierarchy-diagram.svg?sanitize=true"/>

In the diagram, the email service is responsible for managing the template engine and email delivery, while the contacts service has choosen to model each user's contacts as an actor. (This is a very feasible approach in production provided you shutdown actors after a period of inactivity)

Let us focus on the contacts service to see how we can make effective of use of the hierarchy. To support multiple users, we need do three things: 

- Modify our original contacts service to so that we can parameterise its parent and name
- Create a parent to route requests to the correct child
- Add a user id to the path of each API endpoint and add a `userId` into each message.

Modifying our original service is as simple as the following:

```js
const spawnUserContactService = (parent, userId) => spawn(
  parent,
  // same function as before
  userId
);
```

Now we need to create the parent contact service:

```ocaml
let spawnContactsService = (parent) => spawnStateless(
  parent,
  (msg, ctx) => {
    const userId = msg.userId;
    let childActor;
    if(ctx.children.has(userId)){
      childActor = ctx.children.get(userId);
    } else {
      childActor = spawnUserContactService(ctx.self, userId);            
    }
    dispatch(childActor, msg, ctx.sender);
  },
  'contacts'
);
```

These two modifications show the power of an actor hierarchy. The contact service doesn't need to know the implementation details of its children (and doesn't even have to know about what kind of messages the children can handle). The children also don't need to worry about multi tenancy and can focus on the domain.

To complete the example, we finally adjust the API endpoints:

```js

app.get('/api/:user_id/contacts', (req,res) => performQuery({ type: GET_CONTACTS, userId: req.params.user_id }, res));

app.get('/api/:user_id/contacts/:contact_id', (req,res) => 
  performQuery({ type: GET_CONTACT, userId: req.params.user_id, contactId: req.params.contact_id }, res)
);

app.post('/api/:user_id/contacts', (req,res) => performQuery({ type: CREATE_CONTACT, payload: req.body }, res));

app.patch('/api/:user_id/contacts/:contact_id', (req,res) => 
  performQuery({ type: UPDATE_CONTACT, userId: req.params.user_id, contactId: req.params.contact_id, payload: req.body }, res)
);

app.delete('/api/:user_id/contacts/:contact_id', (req,res) => 
  performQuery({ type: REMOVE_CONTACT, userId: req.params.user_id, contactId: req.params.contact_id }, res)
);
```

Now the only thing remaining for a MVP of our contacts service is some way of persisting changes...

## Persistence
[![Remix on Glitch](https://cdn.glitch.com/2703baf2-b643-4da7-ab91-7ee2a2d00b5b%2Fremix-button.svg)](https://glitch.com/edit/#!/remix/nact-contacts-3)

The contacts service we've been working on *still* isn't very useful. While we've extended the service to support multiple users, it has the unfortunate limitation that it loses the contacts each time the machine restarts. To remedy this, nact extends stateful actors by adding a new method: `persist` 

To use `persist`, the first thing we need to do is specify a persistence engine. Currently only a [PostgreSQL](https://github.com/ncthbrt/nact-persistence-postgres) engine is available (though it should be easy to create your own). To work with the PostgreSQL engine, install the persistent provider package using the command `npm install --save nact-persistence-postgres`.  Assuming you've stored a connection string to a running database instance under the environment variable `DATABASE_URL` , we'll need to modify the code creating the system to look something like the following:

```js
const { start, configurePersistence, spawnPersistent } = require('nact');
const { PostgresPersistenceEngine } = require('nact-persistence-postgres');
const connectionString = process.env.DATABASE_URL;
const system = start(configurePersistence(new PostgresPersistenceEngine(connectionString)));
```

The `configurePersistence` method adds the the persistence plugin to the system using the specified persistence engine.

Now the only remaining work is to modify the contacts service to allow persistence. We want to save messages which modify state and replay them when the actor starts up again. When the actor start up, it first receives all the persisted messages and then can begin processing new ones.  

```js

const spawnUserContactService = (parent, userId) => spawnPersistent(
  parent,
  async (state = { contacts:{} }, msg, ctx) => {    
    if(msg.type === GET_CONTACTS) {        
      	dispatch(ctx.sender, { payload: Object.values(state.contacts), type: SUCCESS });
    } else if (msg.type === CREATE_CONTACT) {
        const newContact = { id: uuid(), ...msg.payload };
        const nextState = { contacts: { ...state.contacts, [newContact.id]: newContact } };
      	
      	// We only want to save messages which haven't been previously persisted 
      	// Note the persist call should always be awaited. If persist is not awaited, 
      	// then the actor will process the next message in the queue before the 
      	// message has been safely committed. 
        if(!ctx.recovering) { await ctx.persist(msg); }
      	
      	// Safe to dispatch while recovering. 
      	// The message just goes to Nobody and is ignored.      
        dispatch(ctx.sender, { type: SUCCESS, payload: newContact });            
        return nextState;
    } else {
        const contact = state.contacts[msg.contactId];
        if (contact) {
            switch(msg.type) {
              case GET_CONTACT: {
                dispatch(ctx.sender, { payload: contact, type: SUCCESS }, ctx.self);
                break;
              }
              case REMOVE_CONTACT: {
                const nextState = { ...state.contacts, [contact.id]: undefined };
                if(!ctx.recovering) { await ctx.persist(msg); }
                dispatch(ctx.sender, { type: SUCCESS, payload: contact }, ctx.self);                  
                return nextState;                 
              }
              case UPDATE_CONTACT:  {
                const updatedContact = {...contact, ...msg.payload };
                const nextState = { ...state.contacts, [contact.id]: updatedContact };
                if(!ctx.recovering) { await ctx.persist(msg); }                
                dispatch(ctx.sender,{ type: SUCCESS, payload: updatedContact }, ctx.self);                
                return nextState;                 
              }
            }
        } else {          
          dispatch(ctx.sender, { type: NOT_FOUND, contactId: msg.contactId }, ctx.sender);
        }
    }
    return state;
  },
  // Persistence key. If we want to restore actor state,
  // the key must be the same. Be careful about namespacing here. 
  // For example if we'd just used userId, another developer might accidentally
  // use the same key for an actor of a different type. This could cause difficult to 
  // debug runtime errors
  `contacts:${userId}`,
  userId
);
```



# API

## Functions

### creation

| Method                                   | Returns              | Description                              |
| ---------------------------------------- | -------------------- | ---------------------------------------- |
| `spawn(parent, func, name = auto, options = {})` | `ActorReference`     | Creates a stateful actor. The actor has a processor function with the following signature `('state, 'msg, Context) => 'nextState`  Stateful actors process messages one at a time and automatically terminate if the next state is `undefined` or `null ` |
| `spawnStateless(parent, func, name = auto, options = {})` | `ActorReference`     | Creates a stateless actor. The actor has a processor function with the following signature `('msg, Context) => 'nextState`  Stateless actors process messages concurrently and do not terminate until they are explicitely stopped. |
| `spawnPersistent(parent, func, persistenceKey, name = auto, options = {})` | `ActorReference`     | Creates a persistent actor. Persistent actors extend stateful actors but also add a  persist method to the actor context. When an actor restarts after persisting messages, the persisted messages are played back in order until no futher messages remain. The actor may then start processing new messages. The `persistenceKey` is used to retrieve the  persisted messages from the actor. |
| `start(...plugins)`                      | `SystemReference`    | Starts the actor system. Plugins is a variadic list of middleware. Currently this is only being used with `configurePersistence` |
| `state$(actor)`                          | `Observable<'state>` | Creates an observable which streams the current state of the actor to subscribers. |

### communication

| Method                                   | Returns         | Description                              |
| ---------------------------------------- | :-------------- | ---------------------------------------- |
| `dispatch(actor, message, sender = Nobody())` | `void`          | Enqueues the message into the actor's mailbox. |
| `query(actor, message, timeout)`         | `Promise<'any>` | Enqueues the `message` into the actor's mailbox and waits up to`timeout` milliseconds for a reply. If no reply is received in this time, the promise is rejected. |
| `stop(actor)`                            | `void`          | Stops the actor after it has finished processing the current message. |

### configuration

| Method                                   | Returns | Description                              |
| ---------------------------------------- | ------- | ---------------------------------------- |
| `configurePersistence(persistenceEngine)` | `void`  | Enables the persistence plugin in the actor system using the specified persistence engine. |



## References
Applies to ActorReferences & SystemReferences

| Property | Description                              | Present On     |
| -------- | ---------------------------------------- | -------------- |
| `path`   | The path is the address of the actor. It uniquely identifies the actor in the hierarchy. | Both           |
| `name`   | The name given to the actor. May be automatically generated if not supplied. | ActorReference |
| `parent` | The parent                               | ActorReference |



## Internal Context

#### All Actors

| Property   | Description                              |
| ---------- | ---------------------------------------- |
| `parent`   | `ActorReference` (or `SystemReference`) of this actor's parent |
| `path`     | An object uniquely describing this actor's position in the hierarchy |
| `self`     | The `ActorReference` of the current actor |
| `name`     | The name of the this actor               |
| `children` | A [`Map`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map) containing the references to children of the current actor. The key of the map is the actor's name. |
| `sender`   | The sender of the current message. If left unspecified, this defaults to `Nobody`. Messages sent to `Nobody` are ignored. |



#### Persistent Actors

| Property       | Returns         | Description                              |
| -------------- | :-------------- | ---------------------------------------- |
| `recovering`   | -               | Whether the current messages was previously persisted or is a new message. |
| `persist(msg)` | `Promise<void>` | Saves the message to the event store. Highly recommended that this method is used in conjunction with `await`. |
