title: Creating an online multiplayer web game using Socket.io and Quintus Part 1
date: 2014-07-13 14:17:56
tags:
- Node.js
- Socket.io
- Quintus
- Tutorial
---
One of my coworkers introduced me to [Quintus](http://html5quintus.com/), a young Javascript engine that makes developing simple games a breeze. In this multipart tutorial I'll be showing you how to combine Quintus with Socket.io to make a simple multiplayer game.

## Installation and setup

### Node.js
We'll be using Node.js for our runtime environment. It's super fast to pick up and start developing with, and works great for our multiplayer game. If you don't yet have it installed, go to their [website for the installation link](http://nodejs.org/).

### Getting the proper node modules
Now that we have Node.js installed, let's create the folder where our game will be located.

``` bash
$ mkdir quintus-tag
$ cd quintus-tag
```

Once inside, we'll create a `package.json` file, which will hold information about our Node.js application.

{% codeblock package.json lang:json %}
{
  "repository": {
    "type": "git",
    "url": "git@github.com:mliu95/quintus-tag.git"
  },
  "name": "quintus-tag",
  "author": "Michael Liu",
  "version": "0.0.1",
  "dependencies": {}
}
{% endcodeblock %}

Now to install express and socket.io, which will handle the routing/server and multiplayer respectively.

``` bash
$ npm install --save express
$ npm install --save socket.io
```

By including the `--save` in our install calls, our `package.json` file will automatically update its `dependencies` object.

Now that we have our necessary modules, let's set up our webserver with Express and Socket.io...

<!-- more -->

## Let's get to work!

### Setting up our webserver

Create a file in your directory called `app.js`, and place this code inside.

{% codeblock app.js lang:javascript %}
var express = require('express');
var app = express();
var server = require('http').Server(app);
var io = require('socket.io')(server);
 
app.use(express.static(__dirname + '/public'));
 
app.get('/', function(req, res){
  res.render('/index.html');
});
 
server.listen(80);
console.log("Multiplayer app listening on port 80");
{% endcodeblock %}

Let's go through what this file does.

__Lines 1-4:__ Here we set up our Express server and also our Socket.io engine on top of that server.

__Line 6:__ I plan to include all of our application code inside of a `/public` directory, and instead of inserting `/public` in front of all my files/requires, I tell Express to use `/public` as the home directory

__Lines 8-10:__ Because our game's going to be a single page application, our Express routing will be dead simple. In our case right now whenever we get a request to display whatever is on `localhost:80` (Which is also `localhost`), our app will serve up the `/public/index.html` page (Note that we don't have to include `/public` because of our code in __Line 6__)

__Lines 12-13:__ Now we'll tell our server to listen on port 80, and log a message to the console so we know everything is working just fine.

If we run our server now, we're going to see an error page display complaining about an html module. That's because we haven't yet created our `/public` folder with the `index.html` file inside of it. Feel free to do it now, don't worry about `index.html` being blank. If everything is working you should see a blank webpage! Soon enough, we'll be turning this webpage into our Quintus game.

### Setting up Quintus

Before we start, let's import the javascript libraries we'll be using.

In the public directory, create a `lib` folder which will hold the Quintus engine, RequireJS library, and any other libraries you might want to use in your games.

[Download the Quintus library file here and save it inside the lib folder](http://cdn.html5quintus.com/v0.2.0/quintus-all.js)

[Download the RequireJS library file here and save it inside the lib folder](http://requirejs.org/docs/release/2.1.14/minified/require.js)

Now, let's go into our `index.html` file we created in the previous section and add some script tags to link to our future Quintus game.

{% codeblock index.html lang:html %}
<html>
  <head>
    <script src="lib/quintus-all.js"></script>
    <script src="socket.io/socket.io.js"></script>
  </head>
  <body>
    <script src="/lib/require.min.js"></script>
    <script src="src/game.js"></script>
  </body>
</html>
{% endcodeblock %}

Note: for anyone curious about the Socket.io script link, Socket.io will automatically intercept requests for `socket.io/socket.io.js` and send the appropriate response. [More info here.](http://stackoverflow.com/questions/8689877/cant-find-socket-io-js)

Now let's make a folder called `src` in our public directory and create a new javascript file called `game.js`. This file will house our Quintus engine and communicate with Socket.io.

{% codeblock game.js lang:javascript %}
var Q = Quintus({audioSupported: [ 'wav','mp3' ]})
      .include('Sprites, Scenes, Input, 2D, Anim, Touch, UI, Audio')
      .setup({ maximize: true })
      .enableSound()
      .controls().touch();
 
Q.gravityY = 0;
 
var objectFiles = [
  './src/player'
];
 
require(objectFiles, function () {
  Q.scene('arena', function (stage) {
    stage.collisionLayer(new Q.TileLayer({ dataAsset: '/maps/arena.json', sheet: 'tiles' }));
 
    var player = stage.insert(new Q.Player({ x: 100, y: 100 }));
    stage.add('viewport').follow(player);
  });
 
  var files = [
    '/images/tiles.png',
    '/maps/arena.json',
    '/images/sprites.png',
    '/images/sprites.json'
  ];
 
  Q.load(files.join(','), function () {
    Q.sheet('tiles', '/images/tiles.png', { tilew: 32, tileh: 32 });
    Q.compileSheets('/images/sprites.png', '/images/sprites.json');
    Q.stageScene('arena', 0);
  });
});
{% endcodeblock %}

__Lines 1-5:__ Here we create our Quintus engine. For those curious on what other libraries there are or how this works, I recommend the [Quintus documentation](http://html5quintus.com/guide/core.md#.U8bVQK1dVQ8)

__Line 7:__ Because we're going to be making a top down style tag game, we need to disable gravity.

__Lines 9-11:__ Sourcing our custom Quintus classes that we'll be using in the game.

__Line 14:__ Here we create our main game stage.

__Line 15:__ We use a built in Quintus function to create the walls of our game from a JSON object. [If you need a sample JSON object, here's a download link](arena.json)

__Line 17:__ Creating a new Player class (Which we'll be making later on in this post) and inserting it into the stage with initial coordinates (100, 100).

__Line 18:__ Telling our stage to center the view around the player class.

__Lines 21-26:__ Listing all of the assets we'll need for our game.

__Lines 28-32:__ Here we tell the Quintus engine to load all of our spritesheets and create the `arena` scene we made earlier. I've included the download links for all of the images below:

[/images/tiles.png](tiles.png)

[/images/sprites.png](sprites.png)

[/images/sprites.json](sprites.json)

### Creating the Player class

The last thing we'll have to do to have a basic Quintus game is create our Player class. In your `src` folder create a javscript file called `Player.js`.

{% codeblock player.js lang:javascript %}
require([], function () { 
  Q.Sprite.extend('Player', {
    init: function (p) {
      this._super(p, {
        sheet: 'player'
      });
 
      this.add('2d, platformerControls, animation');
    },
    step: function (dt) {
      if (Q.inputs['up']) {
        this.p.vy = -200;
      } else if (Q.inputs['down']) {
        this.p.vy = 200;
      } else if (!Q.inputs['down'] && !Q.inputs['up']) {
        this.p.vy = 0;
      }
    }
  });
});
{% endcodeblock %}

__Line 2:__ Here we create a new class Player that extends the Q.Sprite class.

__Lines 3-6:__ The `init` function is called whenever we make a new Player class. Notice that we take a `p` parameter, which can be any Javascript object used to set specific properties (Like how we set the initial coordinates for the Player previously). We also can set variables like `sheet`, which defines what sprite our Player will be using, and saves us time from declaring it each time we make a new Player.

__Line 8:__ Here we add some built-in Quintus functionality to our Player class. The `2d` module let's our class obey the law of physics and move around. The `platformerControls` module adds in basic support for moving left, right, and jumping (although we'll be overriding the jumping later). Finally, the animation module will let us play different animations, which we'll be using later to show the difference between the tagged players and the non-tagged players.

__Line 10:__ The `step` function is called by the Quintus engine to update the Player class, and which we'll be using to add some basic functionality to our Player class.

__Lines 11-17:__ Since the `platformerControls` doesn't fully work with our top down style game, we do it ourselves in the `step` function. Quintus by default provides [mapping for the directional keys and more](http://html5quintus.com/guide/input.md#.U8fn661dVQ8), so we can simply look at if the `up` or `down` arrow is being pressed, and adjust the y-velocity property of the Player accordingly. (Note: Any properties of a class are stored within its `p` object).

And that's all there is to it! Now let's run our game server and see if it works:

``` bash
$ cd quintus-tag
$ node app.js
```

Then open up your web browser and go to `localhost:8080`. If everything's working, you should see something like this:

![Whoa!](voila.png)

Boom! Our player can move in all directions and collides with the walls. In the next part of this tutorial, we'll start looking into integrating our Quintus game with Socket.io. [Check it out here!](http://mliu95.github.io/2014/09/16/Creating-an-online-multiplayer-web-game-using-Socketio-and-Quintus-Part2) Thanks for reading!
