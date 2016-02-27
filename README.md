# SHAPERR


## Traffic shaping for Plex Media Server ##

( 2/26/2016: *testable. please report issues.* )

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

NOTE: This method of implementing server side configuration changes is, at best, "mostly reliable" under ideal conditions. Using shell scripts to `tail` a log file, searching for specific conditions (exactly what shaperr does), is the option of least preference over a professionally coded solution using a standard programming language of just about *any* flavor. 

##How do I install it and configure it?

Most commands below will need to be executed as the root user or using `sudo`.

Steps below assume git is already installed
  
1. optional: start your monitoring windows (below) if you haven't already.
2. change directories to /opt
3. `git clone https://github.com/briankelley/shaperr`
4. `cp ./shaperr.conf.orig ./shaperr.conf`
5. edit settings in shaperr.conf
6. as the root user (not sudo) `sh shaperr_start`
7. you can exit root when the prompt returns

Re-running the script at any time (step 6), so long as you do it as the root user, removes throttling for **all currently connected streams** and reinitializes the Plex logfile tail.

New "start" actions will be throttled.


###Monitoring

There's probably better ways to do this, but this works for me. I run each of these commands inside independent putty SSH sessions then tile the windows to suit on a second monitor. You don't need to monitor all of these, but if you're a freak about understanding what's going on, this should help you scratch that itch. For more information about the info presented by these commands, see the man pages for each utility.

- classes
	- `sudo watch -n 1 tc class show dev eth0`
- qdisc's
	- `sudo watch -n 1 tc qdisc show dev eth0`
- filters
	- `sudo watch -n 1 tc -d filter show dev eth0`
- iptables rules
	- `sudo watch -n 1 iptables -t mangle -L OUTPUT -v`
- shaperr logfile
	- `sudo tail -Fq -n 0 /opt/shaperr/shaperr.log`
- CPU / process monitoring (requires "htop")
	- `sudo htop`
	- how my options are configured
	- ![](http://i.imgur.com/fExT80H.png)
	- hit F4 to filter, type `Transcoder`, then `Enter`
	- Only actively transcoded streams are displayed.
- network (requires "iftop")
	- `sudo iftop`
	- These are the options I use... in this order, type: `t`, `t`, `t`, `T`, `S`, `>`, `l`, `32400` (that's an "L")
- disk (requires "iotop")
	- `sudo iotop -P -o -d 2`

###Troubleshooting
- If somehow you end up with rules in your iptables mangle table that are not deleted when a client stops play, are from abandoned clients, or possible due to someone seeking / fast forward rewind, run the script in the app directory titled `SET_IPTABLES_2_DEFAULTS`.
	- **WARNING:** Running this script will wipe out *any* previous configuration you may have applied prior to installing and running shaperr. It's designed to reset IPTABLES to absolute OOB defaults. 

----------
####Known issues

* When "click & drag" seeking occurs using the timeline in a Plex client, the IPTABLES rules may not be added/removed or may, in some cases, be duplicated. This occurs because the Plex server receives a STOP and START request for each position that can be determined on the clients timeline, which, in turn, causes instant execution of server-side IPTABLES commands. By comparison, clicking a specific location on the timeline to seek to results in a single STOP / START event.
* There is currently no clean-up of traffic control classes and filters for clients who disappear as a result of network disconnect, Plex client crash, or closing a client without explicitly STOPPING the video. This lack of clean-up includes pausing a video then closing the client. Videos that complete normally has seen limited testing but appears to function fine. Resolving this will require storing data and running an out-of-process script to kill throttling rules for disconnected clients.
* The shaperr logfile fills with info that is mostly unnecessary for normal operation and some of it is even duplicated. We'll clean this up eventually and make extended logging optional.
* It's possible to have 2 connected clients sharing the same dedicated pipe if the last octet in their IP is the same. See activity below from 2/9/2016.

####Changes / Activity
2/26/2016

* Testing uncovered some play actions, such as direct play / direct stream, not being picked up from the Plex log. String being search for was changed. 

2/9/2016

* Decided to use last octet of IP as basis for `tc` class handle. Randomization of the class handle to avoid duplication collision won't work since there's no IP <-> random number saving ability. The commands that implement traffic shaping are initiated based on data from tailing an active Plex log. Revisit if nosql db implemented at a later date.
* In addition to scripting out the logic, I'm testing the viability of various shaping rules and the extent to which they impact video playability or actually adhere to the desired result. Traffic (packet) shaping is a complex topic with infinite nuance. Determining proper values for `RATE`, `MAXRATE`, and `SERVERMAX` may turn out to be as much "black art" as SEO. YMMV.

####To-do
- [x] temporarily determine `tc` class id naming
- [x] throttle direct play / direct stream "start" actions 
- [ ] research and implement classless queuing discipline sfq/sfb/drr
- [ ] capture network MTU for defined interface and use in `tc` commands
- [ ] implement network exclusion
- [ ] usage & setup instructions
- [ ] implement a nosql db for tracking by session in addition to IP
- [ ] making logging optional
- [ ] auto startup

####Wishlist
- [ ] redesign in Python
