title: Developing on a Myo for MHacks
tags:
  - Myo
  - MHacks
  - SurgeonSimulator
  - Android
  - Wearable
date: 2014-09-28 14:26:20
---

Last week for the hackathon MHacks 2014 I was fortunate enough to snag a [Myo by ThalmicLabs](https://www.thalmic.com/en/myo/) to develop some cool hacks on. Check out the youtube videos below!

{% youtube bU8y1CbwL-s %}  
{% youtube d32xxLw_tP8 %}

## How I did it:

For Myo scripts you can use C++ or Lua scripts for Windows & Mac applications, and Java for making Android apps. Writing Lua scripts and the android application was straightforward and relatively painless. There are event listeners given to you for each gesture and you have access to all rotational data from the Myo. It's a very shallow API so the learning curve was very low.

<!-- more -->

The Myo Surgeon Simulator hack was pretty tricky to get right. Because the Myo only gives the rotational data and not the movement of the hand (as far as I know), I opted to move the game's hand a certain direction whenever my rotation exceeded a certain margin away from the origin. I allowed the user to recalibrate where the origin was by listening for a particular gesture.

Since the Lua script allowed you to send keypresses but not mouse data, I ran a macro program that listened for certain key combinations and gave the mouse a certain acceleration as a result. These key combinations would be sent from my Lua script. I was stumped for an hour or two since mouse movement was working fine, but Surgeon Simulator wouldn't recognize any movements. Turns out the game reads movement based on mouse acceleration and not mouse movement. A short googling and guess + test session yielded the correct macro.

The Myo can tell when you're making a fist, but not when individual fingers are being bent. As a result, when I recognized the fist gesture, I had the Lua script press down all the necessary keys to make the game's hand grab onto objects.

The Android application (apologies for the video quality!) let you map custom movements to a sound. By selecting a sound and performing a specific gesture to indicate the start of the motion, the app would begin storing all rotational information until the aforementioned gesture was made again. From then on, the user could perform a gesture and if it was within a certain threshold of error from the closest mapped motion, the sound associated with that motion was played.

While my data was being correctly stored and motions were being properly compared, the Myo would sometimes shift all of its rotation data. This could be caused by it moving around on my arm or quick jerking motions. The next improvement I would make to my application would be to reset the frame of reference for the rotation whenever the user wanted to perform a gesture. This would make up for any shifts from the original spot.

Overall I had a great time playing around with the Myo. It's an awesome piece of technology and I'm excited to see what people can do with it. Perhaps it could be integrated with the plethora of wearable technology we're seeing coming out?

(Note: I've included the links for the [Myo Soundboard](https://github.com/mliu95/myo-soundboard) and the [Myo Surgeon Simulator](https://github.com/mliu95/myo-surgeonsimulator). Feel free to fork and pick up where I left off!)
