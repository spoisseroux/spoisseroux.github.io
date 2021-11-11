---
layout: post
title:  "[Pd] Didascalies"
date:   2021-09-07 19:48:10
---

**Didascalies remix puredata performance**

<iframe width="560" height="315" src="https://www.youtube.com/embed/DCWmeloQOI4" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

based on Luc Ferrari's Didascalies\
all puredata patches by myself\
samples created and manipulated in ableton live\
this project also plays with the concept of foley on sounds that cannot be extracted from didascalies itself

-----------------------------------------------------------

**main patch**
<img src="https://i.imgur.com/8uswA9C.png">
this patch is the main controller. it allows samples to be triggered, and also allows dynamic setting of interval time (how long before the samples loop again). individual sample volumes can also be adjusted here. the logic for the timing of samples also lives here.

-----------------------------------------------------------

**sample patch**
<img src="https://i.imgur.com/PLIY86l.png">
here samples can be loaded in to be triggered. when the playback finishes the time marker is brought to the beginning to be ready for playback again. when a bang is received from the main patch the sample plays. every sample follows this format from the example string7 patch. 

-----------------------------------------------------------

**piano patch**
<img src="https://i.imgur.com/ET5CfRV.png">
here random midi notes are generated and passed through to ableton live where a piano instrument then outputs. the button in main acts as an on/off switch.

-----------------------------------------------------------

**ableton**
<img src="https://i.imgur.com/MO3Ef0u.png">
midi notes from the drunkenPiano patch are inputted to ableton, where they then go through a dorian mode scale midi effect and then a grand piano instrument.