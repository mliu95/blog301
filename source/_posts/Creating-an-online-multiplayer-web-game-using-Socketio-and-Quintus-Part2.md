title: Creating an online multiplayer web game using Socket.io and Quintus Part 2
date: 2014-09-16 17:55:48
tags:
- Node.js
- Socket.io
- Quintus
- Tutorial
---
In the [previous post](http://mliu95.github.io/2014/07/13/Creating-an-online-multiplayer-web-game-using-Socketio-and-Quintus-Part1/#more), I talked about how to set up Quintus and Node.js for a simple single player web game. Unfortunately, because we're coding up a game of tag, keeping our game single player doesn't make all that much sense.

## Socket.io

### Listening for connections/disconnects

We'll write the code to monitor connections and disconnects on our server-side `app.js`:

{% codeblock app.js lang:javascript %}
...
app.get('/', function(req, res){
  res.render('/index.html');
});
 
var playerCount = 0;
var id = 0;
 
io.on('connection', function (socket) {
  playerCount++;
  id++;
  setTimeout(function () {
    socket.emit('connected', { playerId: id });
    io.emit('count', { playerCount: playerCount });
  }, 1500);
 
  socket.on('disconnect', function () {
    playerCount--;
    io.emit('count', { playerCount: playerCount });
  });
});
 
server.listen(80);
console.log("Multiplayer app listening on port 80");
{% endcodeblock %}

__Lines 6-7:__ Declaring variables that are used to track the number of players and assign a unique id for each player respectively.

__Lines 12-15:__ We wrap these socket emit events in a timeout because if a connection event was fired to the user right when they started connecting to the webpage, they wouldn't have loaded the proper javascript files yet and the event wouldn't do anything. A 1.5 second delay on connection gives a nice buffer time.

<!-- more -->

__Line 13:__ This event, as denoted by `socket.emit`, is only emitted to the user who connected. We can attach a javascript object as a second parameter which is accessible in the client javascript, so any data we want to send to the user from our server should be put here. In this case, we attach the unique id variable for this user.

__Line 14:__ We also emit an event to every single user (Including the one that just joined), which serves to update the player count in the game.

__Line 17-20:__ Here we handle the disconnect logic for the user, simply notifying all users of the app of the new player count.

### Adding a player counter

When a player connects, socket.io will emit a `count` event to all connected users, passing a javascript object with the count of players. Let's create a counter in our webpage that will eventually update on player connect/disconnect.

{% codeblock index.html lang:html %}
<html>
  <head>
    <script src="lib/quintus-all.js"></script>
    <script src="socket.io/socket.io.js"></script>
  </head>
  <style>
    .players {
      z-index: 5;
      position: absolute;
      padding-top: 10px;
      padding-right: 10px;
      right: 0;
      text-align: right;
      font-size: 1.5em;
    }
  </style>
  <body>
    <span id="players" class="players"></span>
    <script src="/lib/require.min.js"></script>
    <script src="src/game.js"></script>
  </body>
</html>
{% endcodeblock %}

__Lines 6-16:__ Set up some CSS styling for our player count span.

__Line 18:__ A span with id `players` that we'll use to update the player count with.

### Adding things on connect!

Let's update our `game.js` file to add a `Player` when a socket.io `connected` event is received.

{% codeblock game.js lang:javascript %}
var players = [];
var socket = io.connect('http://localhost:8080');
var UiPlayers = document.getElementById("players");
...
require(objectFiles, function () {
  function setUp (stage) {
    socket.on('count', function (data) {
      UiPlayers.innerHTML = 'Players: ' + data['playerCount'];
    });
 
    socket.on('connected', function (data) {
      selfId = data['playerId'];
      player = new Q.Player({ playerId: selfId, x: 100, y: 100, socket: socket });
      stage.insert(player);
      stage.add('viewport').follow(player);
    });
  }
 
  Q.scene('arena', function (stage) {
    stage.collisionLayer(new Q.TileLayer({ dataAsset: '/maps/arena.json', sheet: 'tiles' }));
 
    setUp(stage);
  });
});
{% endcodeblock %}

__Line 1:__ Instantiate an array `players` which will hold all of the `Player` objects currently in the game.

__Line 2:__ Create a `socket` object that connects with our server.

__Line 3:__ Declare the `UiPlayers` variable as the element with id `players` we added in `index.html` just now.

__Line 6:__ Declare a `setUp` function that oversees communication with socket.

__Lines 7-9:__ On a `count` event received from the server, we update the text of the `UiPlayers` variable to be the `playerCount` data passed in from the server.

__Line 12:__ Save the `playerId` the server assigned to us.

__Line 13:__ Create a new `Player` object with the properties we passed in. Notice we passed in the socket object declare before.

__Lines 14-15:__ Add our player to the stage and make the viewport follow the player. Notice we moved the logic for actually creating the player and making the viewport follow the player into the socket `connected` event.

__Line 22:__ Call the `setUp` function for our `stage`.

### Sending information back to the server

In order for other people to see my player move, the server must send information about my player to them. In order to receive information about my player, the server needs to receive data from the client. Here's how I do it:

{% codeblock player.js lang:javascript %}
step: function (dt) {
  ...
 
  this.p.socket.emit('update', { playerId: this.p.playerId, x: this.p.x, y: this.p.y, sheet: this.p.sheet });
}
{% endcodeblock %}

Using the socket (we pass it into the player on creation) which we've saved as a property on our player, we emit a socket.io event called `update` to the server. The second parameter of the emit event contains a javascript object where we can tell the server things about our player like position, invincibility status, id of the player, etc.

### Receiving updates from players

Now that our player emits a socket `update` event, we can take that data from our server and distribute it to all other players, so they can see where we move to. I added the following event listener to the server:

{% codeblock app.js lang:javascript %}
socket.on('update', function (data) {
  socket.broadcast.emit('updated', data);
});
{% endcodeblock %}

Now we can write our client-side code to update the other players when we receive their `update` events. I start by creating a new `Actor` class for other players (Note: We can't reuse the original `Player` class we wrote since every instance of the `Player` class is controllable through our keyboard and each instance of the `Player` sends the `update` socket event after every step).

Because the `Actor` class is small, I can just throw it in the `player.js` file.

{% codeblock player.js lang:javascript %}
Q.Sprite.extend('Actor', {
  init: function (p) {
    this._super(p, {
      update: true
    });
 
    var temp = this;
    setInterval(function () {
      if (!temp.p.update) {
        temp.destroy();
      }
      temp.p.update = false;
    }, 3000);
  }
});
{% endcodeblock %}

Notice that save for the interval function, the Actor class has almost no properties set on instantiation or step. This is because we set them ourselves when we get an `update` event with all the necessary properties. This interval function destroys an `Actor` when it hasn't received any update events for three seconds. Without it, if a person were to close out of our webgame the `Actors` would still persist.

Finally, we need to write some functionality that will take our `updated` event data from the socket server and update the corresponding actors. I placed the following in the `setUp` function:

{% codeblock game.js lang:javascript %}
 socket.on('updated', function (data) {
  var actor = players.filter(function (obj) {
    return obj.playerId == data['playerId'];
  })[0];
  if (actor) {
    actor.player.p.x = data['x'];
    actor.player.p.y = data['y'];
    actor.player.p.sheet = data['sheet'];
    actor.player.p.update = true;
  } else {
    var temp = new Q.Actor({ playerId: data['playerId'], x: data['x'], y: data['y'], sheet: data['sheet'] });
    players.push({ player: temp, playerId: data['playerId'] });
    stage.insert(temp);
  }
});
{% endcodeblock %}

__Line 1:__ Declare our socket event listener for the `updated` event.

__Lines 2-4:__ Filter through our `players` array for any players with the same playerId as the one being updated.

__Lines 6-9:__ If we found a player in the `players` array then update their relevant properties.

__Lines 10-13:__ Else, we just create a new `Actor` class for that player, push it onto our `players` array, and insert it into the stage.

### Run it!

![Whoa x 2!!!](seeingdouble.png)

Run your node application once again. You'll see that if you join your server from multiple tabs, there should be players popping up for them! Movement should be updating as well, although for certain browsers like Chrome you need to keep each tab in its own window (Javascript loses focus when it's not the current tab, so it won't be able to update if you're focused on another game tab in the same window). Try getting a family/friend to join the same port and they should be able to run around with you!

In my [next blog post](http://mliu95.github.io/2014/11/14/Creating-an-online-multiplayer-web-game-using-Socketio-and-Quintus-Part3), I'll show you how to get the tagging logic down. Keep an eye out!
