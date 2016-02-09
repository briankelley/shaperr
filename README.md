# SHAPERR

[![Join the chat at https://gitter.im/briankelley/shaperr](https://badges.gitter.im/briankelley/shaperr.svg)](https://gitter.im/briankelley/shaperr?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

## Traffic shaping for Plex Media Server ##

Supported platforms: Ubuntu Linux 12.04+ (*pull req. encouraged*).

##What's the matter with Plex?
* No server-side control over quality settings chosen by Plex client apps.

	Plex client applications determine what video quality will be streamed from your Plex Media Server. The subsequent impact on PMS's upstream bandwidth usage can be substantial depending on where the server is homed.

* Chunked buffering of Plex clients.

	Transcoded videos aren't exactly "streamed" to the client with a smooth, consistent transport such as RTMP. Video is transcoded in chunks then pushed to the client at full wire speed. Like quality settings, this impacts bandwidth usage on the Plex server...

	> So, for a 1Mbps video, it will max out my 4Mbps pipe 25% of the time and spend the other 75% idling. I use the same connection for gaming so lag spikes = death!

##What does SHAPERR do?

* Traffic control / bandwidth throttling on egress (leaving the Plex server) without pleading with users to lower their quality settings.
* Client `RATE` and `CEIL`ing limits. Each connection gets an isolated pipe.
* Server `RATE` and `CEIL`ing limits.
* *Independent* client throttle on PLAY. Throttle removed on STOP. Libraries (thumbs, images, text) viewable at full speed.
* WIP: Network exemptions (e.g. 192.168.x.x)
* WIP: Smooth excess bandwidth capacity distribution to connections that request it (SFQ, SFB, DRR round robin) 


##How do I install it and configure it?
Sadly, I don't have any concrete instructions yet. This is a work in progress and script requirements are in flux. However, this repo is actively being worked.