![](http://d.pr/i/dOVg.png)

# Context

While working out of the Mercer Building in the Startup Hub in downtown Edmonton we were on a floor where the bathrooms were under high demand. There were two single use washrooms, and because of a digital arts school on the same floor at times upwards of 60 people.  This caused an on going issue where you'd end up in line or walking over to the washroom, finding it occupied then returning to your desk only to repeat this process five minutes later. While in this loop you never really got any work done. A better solution had to exist; so, we created one. RasPee Streamer was born!

# Original Idea

Our original spec / idea ended up (as these things very often to) quite different from our final implementation.

The main brain for our Bathroom monitor was to be a RasberryPi. We could easily hook hardware up to it had have it connect to the internet via a WiFi dongle.

Now, we just needed the sensors. Looking online we found lots of options all based around how automation door/window monitoring devices. We wanted them to be wireless so we decided that a z-wave / zigbee / insteon home security switch should serve us well. This however didn't make it into the final implementation.

The more important aspect of our original spec (which we did stick to) was the idea that it should be built as a platform with an available API. A Rails app running on Heroku was to be our main interface point. The Pi (RasPee) should call out to a Rails Server (Streamer) to update it with the current occupied state of the washroom. Using this API we could add wall mounted satus lights washroom notification alerts, mobile apps, etc. The posibilities are endless.

# Implementation

We spent a lot of time wanting RasPee Streamer and a lot ot time complaining to each other when waiting for the washroom. We finally decided to build it.

At our office we had concept of a 'Dev Task', a one day project for two people. The idea behind this was to allow the Engeneering team to prioratize small tasks to make our overall lives better. Looking at a new tech, refactoring a piece of anoying code or in this case, building a live bathroom monitor.

We hadn't placed any orders for components (being the unproactive procrastinating developers we are), which meant we needed to find components we could get immediately from local stores on the same day we were going to build the system.  Some initial online searching against hardware stores for some home automation componenets wasn't looking promising, and eventually we just hit the road to see what we could find.

# Shopping (Goal)

Our first stop on our component scavenger hunt was a hardware store.  We looked through the lighting sections and the electronics sections, but the most promising thing we could find were electronic wireless remotes for lights or garage doors.  We had some ideas for how we could hack these to serve our needs, but they were far from ideal. We moved on.

Our next stop was at [Active Tech](http://www.active123.com/) where we knew we could get a Raspberry Pi and some of it's peripheral components.  At the store we discovered they also carried a competitor to the Raspberry Pi called the [Beagle Board](http://beagleboard.org/) which had some promising specs.  In the end, though, we decided to go with the standard Raspberry Pi Model B we were familiar with and knew could serve our purpose.  We also got lucky looking through the switches that Active had, and found some [Reed switches](http://en.wikipedia.org/wiki/Reed_switch), which are commonly used to detect open doors or windows in security systems.  They weren't wireless, but they would work beautifully to detect the state of the doors through one of the Raspberry Pi's GPIO's.  Beggars can't be choosers, so we decided that the loss of wireless capability wasn't a show stopper and picked up a couple of the Reed switches.  Our shopping list from Active ended up playing out like this:

- 2x Reed Switch
- 1x Raspberry Pi Model B 512 MB
- 1x [PiFace Digital Interface](http://pi.cs.man.ac.uk/interface.htm) for initial prototyping
- 1x Raspberry Pi AC Power Adapter
- 1x USB 802.11n Wifi Adapter
- 1x [Pi Cobbler Breakout Board](https://www.adafruit.com/products/914)
- 1x 4 GB MicroSD Card
- 1x MicroSD to SD Adapter
- 1x Misc. Resistors
- 15' 2 conductor cable (This wasn't enough.  Buy more cable than you need!)  

# Streamer V0.5

In July I spent some time hacking together a prototype of what the Rails server might look like when/if we got around to building the hardware backend for the project.  There were a few key goals:
- Log all state changes of the washrooms to allow for later analysis
- From Day 1 be designed around an API for both modifying and consuming washroom state
- Provide an auto-updating view of the washroom states through a web interface

I decided to use Rails as the framework and Heroku for hosting to simplify getting the server live.  This meant I had to compromise in the implementation of the auto-updating web view as Rails doesn't have support for websockets and I didn't feel running a separate server for that purpose alone justified the additional complexity.

Rails gave me most of the backend infrastructure I needed simply by creating a Washroom resource to wrap the data model for each washroom, and then a basic WashroomState model to track each change to a Washroom.  The Washroom's latest state was calculated by looking at the state of it's most recent WashroomState child object.  This also gave me the chronological logging and history that I wanted.

Both JSON and XML API's for consuming the state's of the washrooms was easily created using the respond_to block functionality available in Rails.

```ruby
class WashroomsController < ApplicationController
  def index
    @washrooms = Washroom.all

    respond_to do |format|
      format.html
      format.xml { render :xml => @washrooms }
      format.json { render :json => @washrooms, :methods => [:state] }
    end
  end
end
```

Hitting the JSON API would return results like the following:

```json
[
   {
      "id":1,
      "name":"Boyz",
      "created_at":"2013-07-15T02:57:43.239Z",
      "updated_at":"2013-07-15T02:57:43.239Z",
      "state":"open"
   },
   {
      "id":2,
      "name":"Girlz",
      "created_at":"2013-07-15T02:57:52.815Z",
      "updated_at":"2013-07-15T02:57:52.815Z",
      "state":"open"
   }
]
```

Simply adding some additional basic controller actions and corresponding routes allowed me to quickly add an API for modifying the washroom states as well:
```ruby
  def open
    @washroom = Washroom.find(params[:id])

    if @washroom.open
      redirect_to action: :index
    else
      render status: 400
    end
  end

  def close
    @washroom = Washroom.find(params[:id])

    if @washroom.close
      redirect_to action: :index
    else
      render status: 400
    end
  end
```

The next challenge was to create a basic web view for directly consuming the state without having to use the API.  I just quickly hacked together a view with two giant boxes that would change from Green to Red depending on the state of the washroom.  I then threw some really basic Javascript polling behind it to auto-update the view if the states changed.  At this point with the view and the API's in place I was able to change the washroom state through direct URL access at http://example.com/washrooms/1/[open|close], and watch the state change on a seperate device in psuedo-realtime.  Win!

![](http://d.pr/i/F3y8.png)

You can view the code for streamer as it was at this release [here](https://github.com/benzittlau/streamer/releases/tag/v0.5).

# RasPee V0.5

With our hardware in hand, it was time to start hacking.  We used the PiFace we had purchased to allow us to get up and running interfacing to our switches as quickly as possible, with the long term goal of removing the PiFace and interfacing directly through the GPIO's to reduce project cost.

The first test was to try and wire up our switches to the PiFace and see if we could get a Python script to detect changes in their states.  We first had to install the library for PiFace following the directions [here](https://github.com/piface/pifacedigitalio), and then I wrote a basic polling loop test script:

```python
import pifacedigitalio
import time

# Basic Initialization Steps
pfd = pifacedigitalio.PiFaceDigital()
pfd.output_pins[6].turn_off()

# Loop and output the value of input pin 0
while True:
  value = pfd.input_pins[0].value
  print value
  time.sleep(1)
```

Using this we were able to get one of our Reed switches working as expected.  The other switch, however, didn't seem to work.  I tried reversing polarities, rewiring, etc, but ultimately we got it working after giving it a nice hard *smack* onto the table.

With our switches proven, it was time to move onto the more ambitious goal of using the Raspberry Pi's inputs directly.  There are some logistics around using a digital input on the Raspberry Pi which the PiFace takes care of for you:

- Pull Up's
- Interrupts
- Debounce

Having the PiFace take care of these for you is *very* nice, and I'd highly recommend using the PiFace to anyone who's simply hacking around or building a prototype:

## Pull Up's
When connecting a switch to a digital input, your first instict would probably be one of these configurations, effectively either connecting the input to a high or low value when the switch is closed:

![](http://d.pr/i/GZOX.png)

The danger of this implementation is that when the switch is open the input is literally connected to nothing.  In this case the value on the wire that leads to the input could vary based on factors as unpredictable as EMI in the air.  This can lead to an input which floats around the trigger value rapidly transitioning between high and low readings; not the behaviour you want when your switch is open.

Many IC's will have integrated pull ups and pull downs, and often even provide functionality to configure the behaviour of this circuitry.  These circuits place a strong resistor between the IC side of the switch and either ground or voltage, to ensure that the value remains at a known state when the switch is open.  Their schematics look like this:

![](http://d.pr/i/6nkR.png)

In the case of the Raspberry Pi Model B, it is left to the user to provide the pull up or pull down circuitry.  In our case we opted for pull-ups, using 1KΩ resistors to 5.0V and grounded the other end of our reed switches, matching the circuit diagram above for a pull-up circuit.


## Interrupts
The PiFace library provides support for interrupt listeners, which are very useful when you want to trigger behaviour based on a change to a pin state.  I wrote some test code to play around with these:

```python
import pifacedigitalio
import time

# Callbacks
def falling_edge(event):
    print "Falling edge detected"

def rising_edge(event):
    print "Rising edge detected"

# Basic Initialization Steps
pfd = pifacedigitalio.PiFaceDigital()
pfd.output_port.all_off()

# Subscribe listeners to the input edges
listener = pifacedigitalio.InputEventListener(chip=pfd)
listener.register(0, pifacedigitalio.IODIR_FALLING_EDGE, falling_edge)
listener.register(0, pifacedigitalio.IODIR_RISING_EDGE, rising_edge)
```

Without the interrupts provided by the PiFace library we instead had to resort to a realtime loop polling for the state of the pins.  Certainly a less elegant solution:

```python
import RPi.GPIO as GPIO
import time

GPIO.setmode(GPIO.BCM)
GPIO.setup(18, GPIO.IN)

# Loop and output the value of input pin 0
while True:
  input = GPIO.input(18)
  print state
  time.sleep(1)

```

## Debounce
Any time a physical eletrical switch closes, there is a lot of noise sent over the wire as the physical mechanics of the switch literally bounce, and the electrical signals spark as the switch wires get close.  When this interfaces into the digital domain, this results in a high frequency set of switching that occurs whenever the switch opens or closes a single time.  To address this it is necessary to implement some form of a 'Debounce'.  Debouncing can be accomplished in the analog circuitry using low-pass filters or, as we did in this case, in firmware.  The goal of a debounce is to try and differentiate true changes in the state of a switch from erroneous noise introduced by the mechanical action of switching.  Our debounce code is one of the simplest forms of a debounce algorithm, and works by rejecting any change of state that occurs less than 0.5 seconds after a previous change.  In our python code this ends up looking like this:

```python
  def update_state(self, new_state):
    if (new_state != self.previous_state and (time.time() - self.last_toggle > 0.5)):
      self.last_toggle = time.time()
      self.previous_state = new_state
      self.update_server()
```


Having addressed the low level problems above necessary for dealing with the Raspberry Pi's GPIO's directly, the rest of the work was fairly straightforward python.  I wrapped the complexity of checking the pin states and firing events to the Rails server into a 'Washroom' class, and then created a basic runtime loop:

```python
import WashroomTools
import time

boys = WashroomTools.Washroom(1, 18)
girls = WashroomTools.Washroom(2, 23)

while True:
  boys.poll()
  girls.poll()
```

# RasPee V1

Now that we had RasPee working on the bench it was time to get it running in the wild. The first step in this process was to install our Reed Switches and get the wires run to them. Luckily both the washrooms were right next to each other and there were some existing conduit lines that we could run our wires next to. The biggest issue we had here was that we soon discovered that we were going to be pretty short on wire for the runs from the switches to the Pi. Luckily we had an extra cat5 cable that worked just fine for the longer run. Since we were still in the experenmental stages everything was mounted with painters tape including the switches just so we could undo this whole process if need be.

![](http://d.pr/i/B4AZ.png)

Our next chalange was positioning RasPee somewhere close to the washrooms and also near power. Conveniently there was some emergency lighting with an extra plug that provided both power and a shelf to set the Pi on.

![](http://d.pr/i/3x9o.png)


# Streamer V1

It had always somewhat chaffed that our incredibly cool platform spanning project was using something as inelegant as polling in the client web interface.  It was functional, but created significant amounts of unnecessary load on the backend, and occasionally introduced delays between an actual state change to a door and when it was reflect in the view.  One option to resolve this would have been setting up a separate backend server on Node to provide sockets for the front-end.  A better option was found, though, in Firebase.  

We had already wanted to port the javascript in the front-end to [Angular](https://angularjs.org/), and a library [already existed](https://www.firebase.com/quickstart/angularjs.html) for integrating Firebase into Angular.  This basically gave us the live updates in the view (without polling!) for free.

A [fairly small set](https://github.com/benzittlau/streamer/commit/ad88387ed0364e8e3d2da9d1279587fb4578f369) of changes were required to change the interface to consume from Firebase, and the backend to push it's changes to Firebase when a state change occured.

With a fairly small amount of work, the performance of the interface improved dramatically, and we no longer had to be embarrased by our javascript polling.

![](http://d.pr/i/7hCE.png)

