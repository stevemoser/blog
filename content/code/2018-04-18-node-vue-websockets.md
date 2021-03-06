---
title: Building realtime apps with Vue and nodeJS
date: 2018-04-18
tags: [development, web, node]
author: anoff
resizeImages: true
---

Wanting to build a simple [SPA](https://en.wikipedia.org/wiki/Single-page_application) as a side project I struggled with things that might annoy a lot of newcomers that do not want to go full vanilla. Which _web framework_, which _style library_, which _server framework_ - and more importantly how does it all work together?

In this post we will put together a bunch of great tools out there to build a realtime web app with a few single lines of code. A quick introduction into the tools that will be used:

- [Node.js](https://nodejs.org/en/): Javascript runtime to build server applications
- [Vue.js](https://vuejs.org/): A webapp framework
- [Material Design](https://material.io/): Set of styled web components by Google using the [vue-material](https://vuematerial.io/) library
- [socket.io](https://socket.io/): Client & server library for websockets
- [servers.js](https://serverjs.io/): An opinionated server framework for Node.js based on [express](http://expressjs.com/)

## Set up a basic Node.js server

![Hello World Node.js](/assets/node-vue-websockets/hello.png)

The first thing we'll do is set up a node server to provide a backend. Using the [servers.js](https://serverjs.io/) library a basic API service can be build with a few lines of code.

```sh
# in any empty directory
npm init # initialize npm project
npm install server
```

Creating a _Hello World_ example:

```javascript
// server.js
// load the server resource and route GET method
const server = require('server')
const { get } = require('server/router')

// get server port from environment or default to 3000
const port = process.env.PORT || 3000

server({ port }, [
  get('/', ctx => '<h1>Hello you!</h1>')
])
  .then(() => console.log(`Server running at http://localhost:${port}`))
```

Running the code with `node server` gives the following output and the website showing _Hello World!_ will be reachable at [localhost:3000](http://localhost:3000)

```sh
Server running at http://localhost:3000
```

For easier development install `npm install nodemon` in the project and change the start command to:

```javascript
// package.json
"scripts": {
  "start": "nodemon -i myapp/ server.js"
},
```

💡 If you're struggling take a look at this [code](https://github.com/anoff/node-vue-websockets/tree/4dc36fc8fac8ee3d179379c0286ee2dfe58f4261) for reference

## Initialize Vue.js project

The easiest way to set up a vue project is to use the `vue`-CLI which is available via `npm install -g vue-cli`. To initialize a project using `webpack` as a bundler run

```sh
vue init webpack myapp
```

Answer the questionnaire with the default question or disable tests you do not want to implement. I chose not to install any test frameworks for this tutorial.

Webpack comes with it's own development server with _hotreloading_ functionality so you see changes in the browser right away. Try it out by starting the server with `npm run dev` (in the `myapp/` directory) and opening the Vue.js template at [localhost:8080](http://localhost:8080)

![Vue.js template project](/assets/node-vue-websockets/vue.png)
> output of the webpack dev server `npm run dev`

![Vue.js template project](/assets/node-vue-websockets/vue-template.png)
> Template Vue.js project at http://localhost:8080

When modifying the Vue.js components the web page will automatically reload

```javascript
// myapp/src/components/HelloWorld.vue

// chnage the script content to
...
<script>
export default {
  name: 'HelloWorld',
  data () {
    return {
      msg: 'Welcome to MY first Vue.js App'
    }
  }
}
</script>
...
```

By simply saving the file the development server will propagate the changes to any open browser window which will automatically reload to

![modified Vue.js template project](/assets/node-vue-websockets/vue-template2.png)

> Modified template with custom message

💡 If you're struggling take a look at this [code](https://github.com/anoff/node-vue-websockets/tree/3e19ae3fd902d719251cf42721ccc83fa27fb394) for reference

## Adding Material Design library

To install `vue-material` run the following command in the Vue.js directory `myapp/`

```sh
npm install vue-material@beta --save
```

Add the following lines to `myapp/src/main.js` to load the `vue-material` components into the app.

```javascript
import VueMaterial from 'vue-material'
import 'vue-material/dist/vue-material.css'
import 'vue-material/dist/theme/black-green-light.css'

Vue.use(VueMaterial)
```

ℹ️ You might have to restart the dev server for this new plugin to take effect

Create a new Vue.js component making use of several `vue-bootstrap` components like the [app](https://vuematerial.io/components/app) container.

```html
<!-- myapp/src/components/Chat.vue-->
<template>
<div class="page-container">
    <md-app>
      <md-app-toolbar class="md-primary">
        <div class="md-toolbar-row">
          <span class="md-title">My Chat App</span>
        </div>
      </md-app-toolbar>
      <md-app-content>
        <md-field :class="messageClass">
          <label>Messages</label>
          <md-textarea v-model="textarea" disabled></md-textarea>
        </md-field>
        <md-field>
          <label>Your message</label>
          <md-input v-model="message"></md-input>
          <md-button class="md-primary md-raised">Submit</md-button>
        </md-field>
      </md-app-content>
    </md-app>
  </div>
</template>

<script>
export default {
  name: 'HelloWorld',
  data () {
    return {
      textarea: "dummy text\nblup\ndummy text"
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.md-app {
  height: 800px;
  border: 1px solid rgba(#000, .12);
}
.md-textarea {
  height: 300px;
}
</style>
```

To load the new component modify the router at `myApp/src/router/index.js`

```javascript
// change HelloWorld -> Chat
import Vue from 'vue'
import Router from 'vue-router'
import Chat from '@/components/Chat'

Vue.use(Router)

export default new Router({
  routes: [
    {
      path: '/',
      name: 'Chat',
      component: Chat
    }
  ]
})
```

![Basic chat app](/assets/node-vue-websockets/chat.png)

💡 If you're struggling take a look at this [code](https://github.com/anoff/node-vue-websockets/tree/99fa39d4761ddaa779192a7f9751820ab7952356) for reference

## Bring in websockets

For the following development the web application will consume from two different endpoints. The `webpack-dev-server` sends the web app sources (HTML, CSS, Javascript) and the node server will supply the `socket-io` endpoint. This is typically not something you want to do in production but since we want both the node server and Vue frontend to be hot reloaded we need two systems - webpack and nodemon.

![development setup](https://www.plantuml.com/plantuml/png/NP0z3i8m34PtdyBgn58KYGKn8JX9sXWef77b12A4k3ik3RzBfBpdptQoZibAElSU0Zl2AbCpsFQ4ZYuOIIua5Tg8hUyef58pyVanue5ZUle95Ty8PmKLG1SIoSwsX8Ua8pxN72E07bWxpg5-vSUg5oeZeNJ3kfwDiHKEB0aNnfWVDKQBMvgbWNZgmc35zdW3n76nZRvhBtmERikU1Q_aFMULLhJFn1glHOhUc_w7VDVJZsTn9D_XEwmfEFtH1m00)

### frontend: vue-socket.io

For the Vue app to communicate with the websocket backend the socket.io library needs to be installed into `cd myApp/`

```sh
npm install vue-socket.io
```

With the node backend running on port `3000` modify your vue application in `myApp/src/main.js` to connect to the backend

```javascript
import VueSocketIO from 'vue-socket.io'

Vue.use(VueSocketIO, 'http://localhost:3000')
```

To bring some very basic functionality into the app we will show messages that were send from other instances in a list and add the ability to send messages.
For sending messages we need to give the `Submit` button an action once it is triggered by adding a `v-on:click` method

```html
<md-button class="md-primary md-raised" v-on:click="sendMessage()">Submit</md-button>
```

The `sendMessage()` function and the socket interactions are specified in the `<script>` tag

```javascript
<script>
export default {
  name: 'Chat',
  data () {
    return {
      textarea: '',
      message: '',
      count: 0
    }
  }, sockets:{
    connect () {
      console.log('connected to chat server')
    },
    count (val) {
      this.count = val.count
    },
    message (data) { // this function gets triggered once a socket event of `message` is received
      this.textarea += data + '\n' // append each new message to the textarea and add a line break
    }
  }, methods: {
    sendMessage () {
      // this will emit a socket event of type `function`
      this.$socket.emit('message', this.message) // send the content of the message bar to the server
      this.message = '' // empty the message bar
    }
  }
}
</script>
```

### backend: socket-io / server.js

Server.js already comems with socket-io bundled into it. The only thing to do in the backend to enable a basic chat operation is to react to a `message` event sent from the UI and propagate this to all connected sockets.

```javascript
// modify server.js to include the socket methods
const { get, socket } = require('server/router')

...

server({ port }, [
  get('/', ctx => '<h1>Hello you!</h1>'),
  socket('message', ctx => {
    // Send the message to every socket
    ctx.io.emit('message', ctx.data)
  }),
  socket('connect', ctx => {
    console.log('client connected', Object.keys(ctx.io.sockets.sockets))
    ctx.io.emit('count', {msg: 'HI U', count: Object.keys(ctx.io.sockets.sockets).length})
  })
])
  .then(() => console.log(`Server running at http://localhost:${port}`))
```

After running `npm start` in the server directory the server will now create logs for every web page that gets opened. It logs the list of currently open sockets.

> Note there is no disconnect event registered yet in this tutorial so the `count` event will only be emitted when a new website connects.

![Server logs for connecting clients](/assets/node-vue-websockets/clients.png)

## Showtime 🍿

Running the demo in two browsers/separate devices in the same network will look like this. It is a very, very, very basic but totally anonymous chat system.

![Very basic chat system](/assets/node-vue-websockets/2chats.gif)

You can find a [repository on github](https://github.com/anoff/node-vue-websockets/commits/master) containing this demo code.

I hope this blog helped you:

1. set up an easy node server
1. bootstrap a Vue project with `vue-cli`
1. get fancy UI elements in Vue using material design
1. integrate websockets to provide realtime communication

What to do next:

- add tests to backend/frontend
- store state/sessions in frontend
- possibly add authentication
- improve UI (e.g. register enter button on message bar)

Feel free to leave a comment or reach out on twitter 🐦