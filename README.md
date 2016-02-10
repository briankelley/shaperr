# SHAPERR

[![Join the chat at https://gitter.im/briankelley/shaperr](https://badges.gitter.im/briankelley/shaperr.svg)](https://gitter.im/briankelley/shaperr?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

## Traffic shaping for Plex Media Server ##

( 2/9/2016: *This is a WIP. not yet testable.* )

Supported platforms: Ubuntu Linux 12.04+

##What's the matter with Plex?
* No server-side control over quality settings chosen by Plex client apps.

	Plex client applications determine what video quality will be streamed from your Plex Media Server. The subsequent impact on PMS's upstream bandwidth usage can be substantial depending on where the server is homed.

* Chunked buffering of Plex clients.

	Transcoded videos aren't exactly "streamed" to the client with a smooth, consistent transport such as RTMP. Video is transcoded in chunks then pushed to the client at full wire speed. Like quality settings, this impacts bandwidth usage on the Plex server...

	> "*So, for a 1Mbps video, it will max out my 4Mbps pipe 25% of the time and spend the other 75% idling. I use the same connection for gaming so lag spikes = death!*"

##What does SHAPERR do?

* Traffic control / bandwidth throttling on egress (leaving the Plex server) without pleading with users to lower their quality settings.
* Client `RATE` and `CEIL`ing limits. Each connection gets an isolated pipe.
* Server `RATE` and `CEIL`ing limits.
* *Independent* client throttle on PLAY. Throttle removed on STOP. Libraries (thumbs, images, text) viewable at full speed.
* WIP: Network exemptions (e.g. 192.168.x.x)
* WIP: Smooth excess bandwidth capacity distribution to connections that request it (SFQ, SFB, DRR round robin) 

##How does it work?

SHAPERR is a collection of Linux shell scripts that, when properly configured and run, will `tail` the active Plex Media Server log file looking for specific information. When SHAPERR sees a new "play" event logged by PMS, it captures the client IP address and the action ("start", "stop") then runs standard `/sbin/tc` commands that establish outbound bandwidth throttling rules in the kernel and iptables firewall for that specific IP.

Clients that attempt to **play** at higher quality / bandwidth settings than those defined in the  `shaperr.conf` will see stuttering video and dropped frames. Lowering quality settings on the client app will allow them to stream smoothly (*all things being equal*). When the Plex client **stops** a video explicitly or plays it through the last second, Plex logs a "stop" event that we use to remove the bandwidth throttling rules and for that IP address.

##How do I install it and configure it?
Sadly, I don't have any concrete instructions yet. This is a work in progress and script requirements are in flux. However, this repo is actively being worked.

----------
####Known issues

* When "click & drag" seeking occurs using the timeline in a Plex client, the IPTABLES rules may not be added/removed or may, in some cases, be duplicated. This occurs because the Plex server receives a STOP and START request for each position that can be determined on the clients timeline, which, in turn, causes instant execution of server-side IPTABLES commands. By comparison, clicking a specific location on the timeline to seek to results in a single STOP / START event.
* There is currently no clean-up of traffic control classes and filters for clients who disappear as a result of network disconnect, Plex client crash, or closing a client without explicitly STOPPING the video. This lack of clean-up includes pausing a video then closing the client. Videos that complete normally has seen limited testing but appears to function fine. Resolving this will require storing data and running an out-of-process script to kill throttling rules for disconnected clients.

####Changes / Activity
2/9/2016

* Decided to use last octet of IP as basis for `tc` class handle. Randomization of the class handle to avoid duplication collision won't work since there's no IP <-> random number saving ability. The commands that implement traffic shaping are initiated based on data from tailing an active Plex log. Revisit if nosql db implemented at a later date.
* In addition to scripting out the logic, I'm testing the viability of various shaping rules and the extent to which they impact video playability or actually adhere to the desired result. Traffic (packet) shaping is a complex topic with infinite nuance. Determining proper values for `RATE`, `MAXRATE`, and `SERVERMAX` may turn out to be as much "black art" as SEO. YMMV.

####To-do
- [x] temporarily determine `tc` class id naming
- [ ] research and implement classless queuing discipline sfq/sfb/drr
- [ ] capture network MTU for defined interface and use in `tc` commands
- [ ] implement network exclusion
- [ ] usage & setup instructions
- [ ] implement a nosql db for tracking by session in addition to IP

####Wishlist
- [ ] python web interface ala "PyPlex"