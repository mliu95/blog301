title: Creating an online multiplayer web game using Socket.io and Quintus Part 3
date: 2014-11-14 19:52:38
tags:
  - Node.js
  - Socket.io
  - Quintus
  - Tutorial
---
Here's a quick demo of our finished game:

<iframe style="width:98%;" height="500" src="http://104.167.115.218:8080/" allowfullscreen></iframe>

[Last post](http://mliu95.github.io/2014/09/16/Creating-an-online-multiplayer-web-game-using-Socketio-and-Quintus-Part2), I showed up how to set up basic Socketio interaction with Quintus to allow for multiple people to join a game of tag. Now, let's work on the final part of our app, which is adding the code to let our players tag each other.

## Player Class

Let's add some properties and event listeners to our player class

{% codeblock player.js lang:javascript %}
Q.Sprite.extend('Player', {
    init: function (p) {
      this._super(p, {
        sheet: 'player',
        tagged: false,
        invincible: false,
        vyMult: 1
      });
      this.add('2d, platformerControls, animation');
 
      this.addEventListeners();
    },
    addEventListeners: function () {
      this.on('hit', function (collision) {
        if (this.p.tagged && collision.obj.isA('Actor') && !collision.obj.p.tagged && !collision.obj.p.invincible) {
          this.p.socket.emit('tag', { playerId: collision.obj.p.playerId });
          this.p.tagged = false;
          this.p.sheet = 'player';
          this.p.invincible = true;
          this.p.opacity = 0.5;
          this.p.speed = 300;
          this.p.vyMult = 1.5;
          var temp = this;
          setTimeout(function () {
            temp.p.invincible = false;
            temp.p.opacity = 1;
            temp.p.speed = 200;
            temp.p.vyMult = 1;
          }, 3000);
        }
      });
 
      this.on('join', function () {
        this.p.invincible = true;
        this.p.opacity = 0.5;
        this.p.speed = 300;
        this.p.vyMult = 1.5;
        var temp = this;
        setTimeout(function () {
          temp.p.invincible = false;
          temp.p.opacity = 1;
          temp.p.speed = 200;
          temp.p.vyMult = 1;
        }, 3000);
      });
    },
{% endcodeblock %}

<!-- more -->

__Lines 4-7:__ Declare some default values for the `tagged, invincible, and vyMult` properties. We'll use `tagged` to tell if a player is tagged or not, `invincible` to tell if a player is invincible, and `vyMult` as a variable to increase the y-velocity of a player.

__Line 11:__ Add some event listeners to our player when we instantiate our class.

__Lines 14-15:__ When a `hit` event is received, check if the player is tagged, the collision object is an Actor (since hit events are triggered on contact with walls too!), the collided player is not tagged, and finally that the collided player is not invincible.

__Line 16:__ Emit a socket.io event `tag` with the `playerId` of the collided player. This lets other players know which player was just tagged.

__Lines 17-22:__ Set the current player's `tagged` state to false. Change their sprite back to a normal player. Give them temporary invincibility. Make them translucent to make it obvious to other players they're invincible. Give them increased speed on the x-axis. Increase `vyMult` to 1.5 which also allows the player to move faster on the y-axis.

__Line 23:__ Set a temporary variable that we can call after a timeout to revert the invincibility and speed increases we gave the current player.

__Lines 25-28:__ Set the player's invincible state to false. Reset their translucent state. Reset their x-speed and y-speed. This is all within a timeout function that executes after 3 seconds.

__Line 33-44:__ When the player first joins the game, we want to give them temporary invincibility and increased speed. This code is copy pasted from the `hit` event.

## Finishing up the server code

{% codeblock app.js lang:javascript %}
var tagged = false;
 
io.on('connection', function (socket) {
  playerCount++;
  id++;
  setTimeout(function () {
    if (!tagged) {
      socket.emit('connected', { playerId: id, tagged: true });
    } else {
      socket.emit('connected', { playerId: id });
    }
    io.emit('count', { playerCount: playerCount });
  }, 1500);
  
  socket.on('disconnect', function () {
    playerCount--;
    io.emit('count', { playerCount: playerCount });
  });
  
  socket.on('update', function (data) {
    if (data['tagged']) {
      tagged = true;
    }
    socket.broadcast.emit('updated', data);
  });
  
  socket.on('tag', function (data) {
    io.emit('tagged', data);
  });
});

setInterval(function () {
  tagged = false;
}, 3000);
{% endcodeblock %}

__Line 1:__ Declare a boolean `tagged` variable that will be used to check if there are any taggers currently in the game.

__Line 7:__ If tagged is false, make the connected player a tagger.

__Line 9:__ Otherwise, simply add them to the game.

__Lines 20-22:__ If we receieve any data from a tagger, we can set tagged to true.

__Line 23:__ The reason we use a `socket.broadcast.emit` instead of an `io.emit` or `socket.emit` here is because `broadcast` will send the event to all sockets OTHER than the emitting socket. This makes sense for the update event because we don't need the server telling us about where our own player is when our client is dealing with our player movement already.

__Lines 26-28:__ Server event for when a player is tagged.

__Lines 31-33:__ Every 3 seconds, set `tagged` to false so that if no tagger is emitting data after 3 seconds, the next connecting player should be made the tagger.

## Finishing the client side game code

Add these last updates to the game.js file

{% codeblock game.js lang:javascript %}
require(['socket.io/socket.io.js']);
 
var players = [];
var socket = io.connect('http://localhost:8080');
var UiPlayers = document.getElementById("players");
var selfId, player;
 
var objectFiles = [
  './src/player'
];
 
require(objectFiles, function () {
  function setUp (stage) {
    socket.on('count', function (data) {
      UiPlayers.innerHTML = 'Players: ' + data['playerCount'];
    });
 
    socket.on('connected', function (data) {
      selfId = data['playerId'];
      if (data['tagged']) {
        player = new Q.Player({ playerId: selfId, x: 100, y: 100, socket: socket });
        player.p.sheet = 'enemy'
        player.p.tagged = true;
        stage.insert(player);
      } else {
        player = new Q.Player({ playerId: selfId, x: 100, y: 100, socket: socket });
        stage.insert(player);
        player.trigger('join');
      }
      stage.add('viewport').follow(player);
    });
 
    socket.on('updated', function (data) {
      var actor = players.filter(function (obj) {
        return obj.playerId == data['playerId'];
      })[0];
      if (actor) {
        actor.player.p.x = data['x'];
        actor.player.p.y = data['y'];
        actor.player.p.sheet = data['sheet'];
        actor.player.p.opacity = data['opacity'];
        actor.player.p.invincible = data['invincible'];
        actor.player.p.tagged = data['tagged'];
        actor.player.p.update = true;
      } else {
        var temp = new Q.Actor({ playerId: data['playerId'], x: data['x'], y: data['y'], sheet: data['sheet'], opacity: data['opacity'], invincible: data['invincible'], tagged: data['tagged'] });
        players.push({ player: temp, playerId: data['playerId'] });
        stage.insert(temp);
      }
    });
 
    socket.on('tagged', function (data) {
      if (data['playerId'] == selfId) {
        player.p.sheet = 'enemy';
        player.p.tagged = true;
      } else {
        var actor = players.filter(function (obj) {
          return obj.playerId == data['playerId'];
        })[0];
        if (actor) {
          actor.player.p.sheet = 'enemy'
        }
      }
    });
  }
{% endcodeblock %}

__Line 3:__ On our client side we'll store all of our other players in an array of Quintus player objects. This will make it easier to update all of the other players when we get information from the server.

__Line 4:__ Set our client socket to listen on port 8080.

__Line 6:__ Here we instantiate the vars `selfId` and `player`, which refer to the `playerId` and `Player` object of ourselves respectively.

__Lines 20-29:__ If our `connected` event is passed in with the `data['tagged']` object property (Look at the app.js we wrote above!), we'll create a new `Player` and set it to a tagger. Otherwise, we'll just create a new `Player`. Note that we set our `selfId` and `player` object to the `Player` created in this function, because the `connected` event is only emitted to our own client. This way we can separate the `Actor` and `Player` classes as we've talked about before and have our keys only control the `Player` class.

__Line 33:__ Have our socket listen for `updated` events, which we wrote above in our app.js code. As we saw from before, the `updated` event will be passed in with a data object filled with all the new properties of another player.

__Lines 34-36:__ Filter through our `players` array that we created before and find the one with the same `playerId` as that passed in by the data object. Since `filter` returns an array and we can assume it only has one `Actor`, we'll just always grab the element at index 0.

__Lines 37-44:__ If our actor was found, we can just update all of its properties with that of the data object.

__Lines 45-48:__ If not, that means that they recently connected and we are just starting to receive update events from them, so we create a new `Actor` and push it into the `players` array.

__Line 52:__ Listen for `tagged` events, which will be emitted to the everyone in the game.

__Lines 53-55:__ If the player being tagged is ourselves, we'll use our `player` reference to easily update our properties.

__Lines 56-62:__ Otherwise, we'll filter through our `players` array like before and update the properties on the appropriate `Actor`.

## Finishing up and deploying on your server

Now that you have everything working, it's time to put this bad boy on the internet!

Once you have the IP of wherever your project will be hosted, you need to change this line of code on the client so that it can connect to our server.

{% codeblock game.js lang: javascript %}
var socket = io.connect('http://104.131.190.55:8080');
{% endcodeblock %}

For this small project, I'm using [Digital Ocean](https://www.digitalocean.com/) to set up a game quickly. Just create a droplet and go into the console. We don't need to install much, just node, npm package manager, and git if you aren't going to do something like ssh/ftp your files over to your server directly.

{% codeblock %}
$ sudo apt-get update
$ sudo apt-get install node
$ sudo apt-get install npm
{% endcodeblock %}

Then switch into your www folder and create your project folder.

{% codeblock %}
$ git init
$ git remote add origin (git url)
$ git pull origin master
$ nodejs app.js
Multiplayer app listening on port 80
{% endcodeblock %}

Well, thanks for joining me on my first blog tutorial! I hope you had as much fun as I did learning Quintus & Socket.io. Also, if at any point in my tutorials I don't fully explain something or you found a better way to do something, by all means comment on my posts and let me know!

