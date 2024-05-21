---
title: "Timeflip Server"
---
# Project Spotlight: Timeflip Server

**The Device**
The [TimeFlip2](https://timeflip.io/) is twelve-sided bluetooth dice about the size of your fist. Its intended use is for time tracking, and it's primarily targeted towards freelancers who may want to keep precise track of their hours worked on a specific project. Each side of the dice is assigned a task or an activity, and a timer is started when that side is faced up.

I first became interested in the device as an easy way to keep track of the amount of time I spend practicing guitar. It's useful to know that I'm putting in, say, thirty minutes of practice each day. But manually starting a timer or having to note start and end times was a hassle. The TimeFlip2 presents a simple and easy solution.

The only real sticking point is that this data gets collected through a companion mobile application and then stored on the TimeFlip company's own servers. If you fall into either the "data-hoarder" or "privacy-enthusiast" genus of internet dweller, this may be a red flag.

What sets the TimeFlip2 apart from the Timeular tracker, a similar device which is also more expensive despite having fewer sides, is that the TimeFlip2 has an open-source bluetooth low-energy protocol.

Or, at least, semi-open source. There's a couple of Markdown files on the company's Github that gives a pretty good explanation of how the devices attributes work, which is leaps and bounds better than most smart devices.

**The Library**
This means that creating my own software to exploit the TimeFlip was within the realm of possibility-- though, it would be nice to not have to start from scratch. After a bit of research, I was able to find a Python repository called [pytimefliplib](https://github.com/pierre-24/pytimefliplib), which appeared to implement the TimeFlip2 protocol. Perfect!

Except for one minor issue. At the time, pytimefliplib supported version 3 of the protocol. My TimeFlip used version 4. There were a handful of incompatible changes between the protocol versions, but the core functions-- connecting to the device and querying the states of its sides, or *facets* as they are called in the documentation-- was the same.

What is a developer to do but update the library to support version 4 themself and open a pull request with the changes? That stage of this project was completed about a year ago. Then I got busy with other projects-- shame I wasn't able to track the time I spent on them!-- before I was able to get back to this one.

**The Project**
Now that I had a Python library to interact with the TimeFlip device, I had the tools necessary to write an application that could actually record events from it. I knew I wanted to use [Grafana](https://grafana.com/) to display the data that was collected, so I had no interest in building a web application. I just needed a daemon-style program that would connect to the TimeFlip using pytimefliplib and then store events in a database. 

As with all things, there was a bit of a learning curve. When all was said and done, I had learned a fair amount about Python bluetooth libraries-- one of the features I knew I wanted was a way to specify the specific bluetooth adapter to be used for the connection to the TimeFlip, which took me down a rabbit hole into the `bluez` project for Python. 

I also learned a surprising amount about Grafana. For example, I figured out how to make it so the colors on the timeline graph used to display activities would match the colors assigned to each facet on the TimeFlip device itself, which was pretty cool.

And, probably my favorite feature, I added a disco mode, so that the colors of the TimeFlip are chosen randomly.

If you're interested in using the TimeFlip server project for yourself, the source code is here, and the Docker image is here. The official `pytimeflip` library is here, and my personal fork is here.

Thanks for reading, and have a nice day. 😎 