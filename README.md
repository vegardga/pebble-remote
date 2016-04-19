#How I turned my Pebble into a TV-remote
I love playing around with new technologies. As a software developer with an interest in robotics, the pleasure of creating and working with both software and hardware is indescribable. My last project was therefore a dream come true when I used a smartwatch to control different devices in our home, including the television.

##The idea
At home we currently use at least three different remotes when watching television. It results in a daily battle of finding the remotes, picking them up, and pressing some buttons on each of them, one at a time. A classic first world problem, I know. Instead of buying a universal remote controller I decided to build my own. 

James Bond was, and still is, one of my favorite fictional secret agents. The things he manages to do from various gadgets are always one step ahead of its time, and sometimes way into the future. Agent 007 can use his cell phone to remotely control cars, take out bad guys with explosive pens, or even cut metal with a laser watch. Why not take inspiration from this, and create something that could fit right into the next Bond movie?

I got a Pebble Steel last Christmas and it quickly became a part of my daily life. I constantly receive e-mail and text message notifications throughout the day. The ability to take a quick glance at my wrist instead of using my smartphone to check if it needs my attention immediately is priceless. The Pebble is always with me, and it's always right there on my wrist. So, why not use the Pebble to handle the interaction with the television?

There are probably a myriad of different ways to turn a smartwatch into a remote control. Personally, I prefer to mostly use some of the stuff I have laying around and then include some new technology when starting a project. Infrared sensors and wireless communication between a Raspberry Pi and an Arduino became this projects biggest challenge, but the achievement was well worth it. 

##What's needed
I ended up using the following devices and sensors for the Pebble remote. Pebble, iPhone, Raspberry Pi, and Arduino. 433 MHz radio transmitter and receiver, IR transmitter and receiver, and some jump wires and resistors. Mix in some imagination and determination, and you're all set!

##IR
Infrared, or IR, communication is an inexpensive and widely used wireless communication technology. IR has a slightly longer wavelength than visible light, and are therefore perfect for wireless communication since it's undetectable to the human eye. Most televisions these days still use IR for interaction, and the remote is easily replaceable by an Arduino with transmitting IR LED attached.

When pressing a button on your favorite remote, the IR LED repeatedly blink thousands of times per second to transmit information to the TV. Before transmitting IR signals from the Arduino, you have to find the correct signal frequency to broadcast. Sounds like an interesting challenge? Not really. Luckily Ken Shirriff has written a great Arduino library for sending and receiving IR data [1]. Simply attach an IR receiver to the Arduino, press wanted buttons on the different remotes, and record the received data. Easy as pie!

The next step is replacing the IR receiver with a transmitter and slightly change some parts of the code from receiving to sending signals. To complete the Arduino you need a way to tell it when to blink the IR transmitter. This is easily done by attaching a 433 MHz radio receiver to the Arduino, and using an additional Arduino library for operating remote controlled devices through radio signals [2]. A fully functional program could look something like the code below. 

```c
#include <IRremote.h>
#include <RCSwitch.h>

unsigned int recorded_tv_ir_signals[37] = {2550,950,400,950,400,450,400,500,1200,1400,400,500,400,500,750,1000,400,450,400,500,400,450,450,450,400,500,400,450,400,500,800,500,400,950,400,450,400};
unsigned int recorded_sound_ir_signals[37] = {2500,1000,400,900,400,500,400,450,1250,1400,400,450,400,500,800,950,400,500,400,450,400,500,400,500,400,450,400,500,800,950,400,500,400,450,400,500,400};

IRsend irsend;
RCSwitch mySwitch = RCSwitch();

void setup() {
  mySwitch.enableReceive(0);
}

void loop() {
  if (mySwitch.available()) {
    int receivedValue = mySwitch.getReceivedValue();
    if (receivedValue == 1) {
      irsend.sendRaw(recorded_tv_ir_signals, 37, 38);
    } else if (receivedValue == 2) {
      irsend.sendRaw(recorded_sound_ir_signals, 37, 38);
    }
    mySwitch.resetAvailable();
  }
}
```

##Raspberry Pi as the middleman
I already had a Raspberry Pi mounted on the wall, running a Node.js server and displaying information like the weather and upcoming calendar events. A perfect device for communicating with the IR transmitting Arduino! The choice for communication between the Raspberry Pi and Arduino fell on 433 MHz radio signals. Why not use a Wi-Fi shield on the Arduino you may ask? Because I can! A perfect chance to try another communication protocol and learn something new.

The 433 MHz transmitter connected to the Raspberry Pi through GPIO pins is easily controlled through Python scripts or command line utilities. Node.js can handle both, so with a small REST API the transmitter could be controlled from any device connected to the local network. Now, any network request to the Raspberry Pi is forwarded to listening devices through radio signals.

433Utils is a collection of code and documentation designed to assist in the connection and usage of 433 MHz transmit and receive modules, an open source project available on GitHub [3]. The Raspberry Pi can broadcast radio signals with just a few lines of code using this command line utility, here illustrated with a Python script.

```python
#!/usr/bin/python
# -*- coding: utf-8 -
import sys
import os
os.system("sudo ./../433Utils/RPi_utils/codesend " + str(sys.argv[1]))
```

A lightweight server on top of Node.js does not have to be any more advanced or complex than these few code lines.

```javascript
var express = require('express');
var app = express();

app.get('/:tvcode', function (req, res){
  var options = {
    mode: 'text',
    pythonPath: '/usr/bin/python',
    pythonOptions: ['-u'],
    scriptPath: '<path to python script>',
    args: [req.params.tvcode]
  };
  PythonShell.run('tv-remote.py', options, function(err, results){
    if(err){
      console.log('Error in pythonshell!');
      // handle error
      return;
    }
    console.log(results);
    // handle success
  });
  return res.send("Sending radio signals with code " + req.params.tvcode);
});

var server = app.listen(3000, function () {
  var host = server.address().address;
  var port = server.address().port;
  console.log('Listening at http://%s:%s', host, port);
});
```

##Pebble.js
Pebble has given developers the opportunity to write Pebble applications through JavaScript rather than C code. Pebble.js is still in beta at the time of writing, but it's more than capable already for writing minor applications with web interactions. 

Pebble.js provides an ajax library for connecting smartwatch applications to the Internet, and after going through a couple of tutorials my Pebble could say hello to the world. I was now able to control the television from my wrist. A truncated code example follows.

```javascript
var UI = require('ui');
var ajax = require('ajax');

var main = new UI.Card({
  title: 'TV-remote',
  subtitle: 'How to:',
  body: 'select - turn on/off'
});
main.show();

main.on('click', 'select', function(e) {
  var card = new UI.Card({
    title:'On/off',
    subtitle:'Performing action'
  });
  card.show();

  ajax({
    url: '<url to rest api>',
    type: 'json'
  },
  function(data) {
    console.log('Success!');
    // handle success
    card.hide();
  },
  function(error) {
    console.log('Failed fetching data: ' + error);
    // handle error
    card.hide();
  });
});
```

##Only the beginning
One of the great things with using the Raspberry Pi as a server is the possibility for using other devices as additional controllers. The Pebble is only the first step. Other possibilities include Android wear and the newly launched Apple Watch. The opportunities are endless, and are not strained to smartwatches. A cliché, but still, the only limitation is your imagination. Every device connected to the local network is potentially a remote controller for the television.

Finally, to sum up the flow and communication chain. The Pebble sends ajax requests to a smartphone over Bluetooth. The phone forward these requests to the Raspberry Pi, which broadcasts radio signals based on the received request. The Arduino receives these radio signals and forwards them through IR signals to the television. Good technology is indeed indistinguishable from magic!
 
That's it. That's one way turning your smartwatch into a TV-remote. Fun for you and me to build, easy for others to use.

* [1] - https://github.com/shirriff/Arduino-IRremote
* [2] - https://code.google.com/p/rc-switch/
* [3] - https://github.com/ninjablocks/433Utils