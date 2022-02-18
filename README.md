# NetHole.3beta
********************
*** INTRODUCTION ***
********************

      NetHole is a program that creates a tarpit or, as some have
      called it, a "sticky honeypot".  NetHole takes over unused IP
      addresses on a network and creates "virtual machines" that
      answer to connection attempts.  NetHole answers those connection
      attempts in a way that causes the machine at the other end to
      get "stuck", sometimes for a very long time.

---
--- How does it work?
---

      NetHole works by watching ARP requests and replies.  When the pgm
      sees consecutive ARP requests spaced several seconds  apart,  without
      any  intervening  ARP  reply,  it  assumes that the IP in question is
      unoccupied.  It then "creates" an ARP reply with a bogus MAC address,
      and fires it back to the requester.

      An example (from a tcpdump of NetHole running on my network):
	14:18:28.832187 ARP who-has xx.xx.xx.13 tell xx.xx.xx.1
	14:18:29.646402 ARP who-has xx.xx.xx.13 tell xx.xx.xx.1
	14:18:31.707295 ARP who-has xx.xx.xx.13 tell xx.xx.xx.1
	14:18:31.707574 ARP reply xx.xx.xx.13 is-at 0:0:f:ff:ff:ff

      There is no xx.xx.xx.13 machine on my network.  In this case,
      the timeout was set to 3 seconds (it's a command line
      parameter), and when that final "who-has" came in, the "is-at"
      reply that you see was generated by NetHole.

      There isn't a MAC address of 0:0:f:ff:ff:ff either.  It doesn't
      exist.

      But now, the router (xx.xx.xx.1) believes that there some
      machine at xx.xx.xx.13, and that it resides on the MAC address
      0:0:f:ff:ff:ff, and so it dutifully sends packets on.  In
      essence, we've created a "virtual machine" on that IP address.

      Now, NetHole also watches for TCP traffic destined for the ether
      address 0:0:f:ff:ff:ff.  When it sees an inbound TCP SYN packet,
      it replies with a SYN/ACK that "tarpits" that connection
      attempt.  Everything else is ignored. (Well...  sort of.  NetHole
      also tries to give its "virtual machines" some character...  you
      can ping them, and they respond to a SYN/ACK with a RST...)

      There's more to it than that (obviously...) but you'll need to
      read further.


************************
*** How do I run it? ***
************************

 Glad you asked!

The short answer:

	Usage: NetHole <options> <BPF filter>



The long answer:

---------------------------------------------
-- Interfaces / IP address / Netmask / BPF --
---------------------------------------------


--device (-i) interface			: Set a non-default interface

      If your machine has more than one interface, and NetHole choses
      the "wrong" one, you can use this option to direct it to the
      correct one. Use a device name ("eth0") as a parameter.

      On Windows, you can use the "-D" parameter to see the list of
      interfaces is recognized.


--quiet (-q)				: Do not report odd
					  (out of netblock) ARPs

      If you have two netblocks, with non-contiguous addresses, NetHole
      will complain about seeing ARPs that it believes it shouldn't
      see.  This will tell it to "be quiet."


--bpf-file (-F) filename	        : Specify a BPF filter filename

      Designates the name of a file containing a BPF filter pointing
      to machines/ports to add to the tarpit.

      Note that connections specified by the BPF filter will also be
      tarpitted.

      As with the command line BPF filter, these connections MUST be
      firewalled to DROP inbound traffic or this won't work!


--mask (-m) mmm.mmm.mmm.mmm		 : User specified netmask
--network (-n) nnn.nnn.nnn.nnn[/nn]	 : User specified network number

      Normally NetHole picks up information from the interface in order
      to determine the capture subnet.

      Sometimes you might run NetHole on an unconfigured interface (one
      without an assigned IP address). In this case, you'll have to
      provide the "netmask" and the "network number" for the capture
      subnet.

      The "n" parameter accepts a CIDR-format address. So the class C
      subnet 192.168.99.xx could be specified as:

      -n 192.168.99.33/24

      or as:

      -n 192.168.99.33 -m 255.255.255.0

      Note: KNOW WHAT YOU'RE DOING. If these numbers are not correct,
      BAD THINGS may happen.


--my-ip-addr (-I) nnn.nnn.nnn.nnn	 : Specify system's IP address
--my-mac-addr (-E) xx:xx:xx:xx:xx:xx	 : Specify system's MAC address

      NetHole needs to know the NIC's (Network Interface Card) IP and
      MAC addresses. This information is used to construct the ARPs
      that NetHole uses to determine if an IP address is occupied or
      not.

      On certain systems (e.g. old Win98), the API that supports the
      libdnet intf rtn is not installed. So you have to specify the
      system IP and MAC address yourself.

      If you specify one, then you have to specify the other. You will
      also have to manually specify the capture subnet information.

      Note: As mentioned above, be VERY careful when manually
      specifying this information. If you get it wrong, bad things can
      happen.


--------------------------
-- Tarpitting behaviour --
--------------------------

--throttle-size (-t) datasize		: Set connection throttling
					  size in bytes

      Since you're "inviting" the scanners in, you might as well place
      some restrictions on them.  This option sets the TCP window
      advertisement to limit the amount of data sent by the scanner.
      The number of data bytes to allow per packet is passed as a
      parameter.

      Default value is 10 unless persist mode is used. With persist
      mode, defualt value is 3.

--arp-timeout (-r) rate			: Set arp timeout rate
					  in seconds

      The number of seconds to wait between arp requests before
      deciding that an IP address is unused.

      Here is a description of the algorithm used:

      On an IP by IP basis, we store a time and an originating IP
      address:

      1) When you see an ARP request, check the current time:

            a) If currently stored time is 0 or the arp comes from a
               different address than the one stored, store the
               current time and the requesting IP and return.

            b) If the stored time is less than "-r" seconds ago,
               ignore it and return.

            c) If currently stored time is more than a minute ago,
               store 0, return. (Max timeout)

            d) Otherwise, grab the IP!

      2) See an ARP reply, set stored time to 0.

      The default timeout is 3 seconds.



------------------
-- IP Capturing --
------------------

--switch-safe (-s)		  : "Safe" operation in a 
				    switched environment

      Under a switched environment it is possible for LaBrea to see an
      ARP request, but not see the resulting ARP reply.  LaBrea can
      still work under these conditions by sending out "mirror" ARP
      requests of its own.

      If this parameter is specified, when LaBrea sees an inbound ARP
      request the pgm sends out a duplicate ARP request for the same
      IP, with the LaBrea server itself as the target for the reply.


--exclude-resolvable-ips (-X)	   : Automatically exclude resolvable
				   IPs from capture.

      On startup, this will attempt DNS resolution on all IPs within
      your netblock, and automatically exclude any that resolve.


--disable-capture (-x)		   : Disable IP capture
  
      This instructs LaBrea NOT to capture IPs.


--hard-capture (-h)		   : "Hard" capture IPs

      The -h option instructs LaBrea that once it captures an IP
      address, it needn't wait for a "-r" timeout the next time it
      sees an Arp request for this IP.  IPs are "hard" captured.

      See the section on the configuration file for further
      information.


--soft-restart (-R)		    : Wait while recapturing
				      active connects

      "Soft Restart" mode. What this does is to hold off on any new
      attempt to force incoming sessions into the "persist" state for
      5 minutes. This lets things settle down and gets the bandwidth
      calculations going. Note that during this period, IPs will still
      be captured, pings will still be responded to (if specified),
      etc.
  

      After I changed some stuff in LaBrea, I thought I would be
      tricky and restart LaBrea quickly so I could keep hold of the
      connections I had already trapped.  And lo, one of the dogs of
      the internet chose that moment to hit me with a scan.  LaBrea
      didn't have enough information for correctly calculating
      bandwidth yet, so I ended up with *WAY* too many connections.
      
      "Soft restart" mode prevents this from happening.


--auto-hard-capture (-H)           : Automatically hard capture
				     addresses not excluded.

      This marks all non-excluded and all non-hardexcluded IPs as
      being hard captured.  Use CAREFULLY.


------------------------------
-- Persistent state capture --
------------------------------

--max-rate (-p) maxrate		    : "Persist" state capture
				      connect attempts

      LaBrea will permanently capture connect attempts within the
      limit of the maximum data rate specified (in Kbits/sec).

      This value is expressed in KiloBytes/Sec. (This is a change from
      previous versions.)

      This is UNBELIEVABLY COOL (if I do say so myself...)  If you
      specify this flag and a maximum bandwidth, several things will
      happen.

      First of all, this forces data throttling to 3 bytes (see the
      "-t" option above).
   
      Then, when a connection is attempted, LaBrea will force the
      connection into what is known as "persist" state.  In persist
      state, the connection NEVER times out.  You'll literally hang
      onto the scanning thread until you stop or they stop.

      Running unchecked, this could have a detrimental effect on your
      bandwidth, so LaBrea will make every effort to limit itself to
      the maximum bandwidth that you specify (in Kb/sec).  If it
      can't capture a connection, LaBrea will still tarpit it.  Note:
      It'll stay pretty NEAR your MAXBW number... YMMV.

--log-bandwidth (-b)		      : Log bandwidth usage to syslog
   
      This will send an update on the current bandwidth being consumed
      by the -p option to the log every minute.  If you're
      interested...  (Note: it'll only work if you have -p enabled.
      Duh...)

--persist-mode-only (-P)	      : Persist mode capture only.

      Persist mode capture only.  This tries to limit bandwidth by
      only persist capturing.  When we're at full bandwidth, standard
      tarpitting won't happen, but because the same "conversation"
      that leads to persist capture also has the side-effect of
      tarpitting, when we're below our set bandwidth, it's not really
      effective. (It was easy to do though...)


-------------------------------
-- Virtual machine behaviour --
-------------------------------

--no-resp-synack (-a)		      : Do not respond to SYN/ACKs
					and PINGs

      By default, LaBrea's "virtual machines" will respond to a
      SYN/ACK packet with a RST.

      This is nice behavior, because it makes it difficult for people
      to use your empty IP addresses to "spoof".

      The virtual machine will also respond to a ping, which acts as
      an invitation to anything that preceeds a scan with a ping to
      see if the target exists.  Like say... NMap, or most worms.  If
      you DON'T want this behavior, use the "-a" option to disable it.

--no-resp-excluded-ports (-f)           : "Firewall" excluded ports.

      The -f parameter says to "firewall" excluded ports.  With this
      option, excluded ports will act as if they were firewalled to
      DROP inbound connections.

      The result is that nmap scans of LaBrea virtual machines in the
      capture subnet will take a long time. This discourages hacking
      activity while at the same time generating log entries that warn
      you of the activity.

      LaBrea is automatically configured to always respond to the
      "usual" hacking ports.
      
      Also, if there is enough activity on some other port, then the
      virtual machines will adjust by starting to respond to incoming
      connections on this new port.

      ----------------------------

      Before giving the detailed explanation, first some definitions:

      a) A standard port is one that LaBrea always responds to. These
      ports are the ones that hackers and worms look for (e.g. telnet,
      http, ftp, etc). See ctl.c for the complete list.

      b) An excluded port is one that has been configured as such in
      the configuration file. Even a standard port can be forced to be
      excluded.

      c) A dynamic port is one that is neither standard, nor excluded.

      ----------------------------

      When "-f" is specified, LaBrea behaves as follows:

      1) Excluded ports will do not respond at all (DROP).  

      2) Activity on a standard port will be handled as
      usual (i.e. tarpitting, persist mode)

      3) If LaBrea sees activity on a dynamic port, then it starts
      counting the number of SYNs received (ie incoming
      connections). When there is enough activity on the port, then
      LaBrea will start responding to incoming connects:
      
	a) If SYN count is less than 6, then drop the incoming
           connection, but increment the counter by 1.

        b) If SYN count is 6 or more, then respond to the incoming
           connection (tarpitting / persist mode).

      4) Every 15 minutes, all port counts < 255 are reduced by one to
      eliminate the effect of SYN "noise". However, once a port count
      reaches 255, the port will always respond to incoming SYNs.


--no-arp-sweep			 : Don't perform initial arp sweep of
				   capture subnet

      LaBrea has a number of safety mechanisms built-in to avoid
      causing problems with its virtual machines.

      By default, LaBrea will do an arp sweep of the capture subnet in
      order to detect IPs that are already occupied by active
      machines.  Arps are generated in bursts of 85 at 2 minute
      intervals. However if the capture subnet is too large (>1024
      addresses), then a warning message is given, and the arp sweep
      is turned off.

      Specifying this parameter means that LaBrea will not do the
      initial arp sweep.


--------------
-- Logging ---
--------------

--log-to-syslog (-l)		: Log activity to syslog

      Sends all messages to system syslog once the initialisation is
      completed. This is the default behaviour on Unix systems.

--verbose (-v)			: Verbosely log activity to syslog

      Log all IPs "captured", IPs "teergrubed", plus all activity from
      the "teergrubed" hosts.

      Specify twice for more effect.


--log-to-stdout (-o)		 : Output to stdout instead of syslog

      This sends log information to stdout rather than to syslog.
      This option also implies and sets the -d option (Do NOT detach
      process).

      Yes, I know... LaBrea is chatty and dumps a whole lot of stuff
      into syslog.  This gives you the option to have LaBrea log
      information go to stdout instead.

      "-o" is the default behaviour on Windows systems.


--log-timestamp-epoch (-O)       : Same as -o w/time output in
				   seconds since epoch

      The same as the "-o" option, but formats the time stamp
      differently to make it easier for other "logfile analysis"
      programs to parse it.


--version (-V)			 : Print version information and exit


---------------------------------
-- Windows-specific parameters --
---------------------------------

==> Note that on Windows systems, messages are sent by default to
    stdout. Also LaBrea is not yet able to detach itself and run as a
    standalone Windows service. 

==> The following parameters are specific to Windows systems only:


--list-interfaces (-D)		 : List available interfaces

      LaBrea uses two different APIs to interact with the NIC (network
      card): libdnet and WinPcap. The libdnet intf API is used to
      extract information from the NIC and to generate packets. The
      WinPcap API is used to sniff.

      Unfortunately, these two APIs have different nomenclatures for
      the same underlying NIC.

      Specifying this parameter causes LaBrea to generate the list of
      available interfaces. Both the WinPcap and the libdnet device
      lists are given.

      In each list, the interface by default is indicated.

      You get to pick an interface from each list if the defaults are
      not right. Use the "-i" parameter (see above) to pick the
      libdnet interface. See the following parameter for the WinPcap
      device.


--winpcap-dev (-j) nn		 : Specify WinPcap device 

      The WinPcap device driver is used for packet sniffing.

      By default, the first device in the WinPcap list is the one that
      is used.

      The "-j" parameter can be used to specify another entry in the
      list.

      For instance, "-j 2" says to use the 2cd entry in the WinPcap
      device driver list.

      ----------------------------

      Note: It is ESSENTIAL that the -j and -i parameters specify the
      SAME physical interface (NIC).
       

--syslog-server nnn.nnn.nnn.nnn		: Specify address of remote
					  syslog server
--syslog-port nnn			: Specify port to be used for
					  remote syslog

      On Windows systems, LaBrea offers syslog support.

      For Windows NT and up, log messages will be sent to the local
      Windows Application Event log if the "-l" parameter is
      specified.

      However, when "--syslog-server" is specified as well, then the
      pgm will send log messages to a remote syslog server. This will
      work even on Windows 98 or ME systems.

      Finally, if the remote syslog doesn't open for some reason, then
      LaBrea will fail over to the local application Event log.


--------------------------------
-- Special modes of operation --
--------------------------------


--dry-run (-T)				: Test mode - Prints out debug
					  info but DOES NOT RUN

      Test mode. If you're having trouble, try this first and see if
      LaBrea is picking up the information on your adapter, netblock,
      netmask, etc...  correctly.  This prints diagnostic information
      and then exits.

--foreground (-d)			: Do NOT detach process.

      Some people want to run LaBrea under the control of another
      process. This keeps LaBrea from detaching and running as a
      daemon.

      This is the default (only!) behaviour on Windows systems.


--usage --help (-?)			: Give help messages


--no-nag (-z)				: Turn off nag message

      ==> IMPORTANT ==> Be sure that you read the "Potential Issues"
      section in the INSTALL documentation before you actually use
      LaBrea.

--init-file filespec			: Specify alternative location
					  for the configuration file

      By default, LaBrea looks for the configuration file as follows:
      
      Unix systems	/usr/local/etc/labrea.conf
      Windows systems	LaBrea.cfg in the current execution directory

      The "init-file" parameter can be a full filespec complete with
      path information. LaBrea will look in the specified location for
      the configuration file.

--debug nn				: Produce debugging output

      If debugging is compiled into LaBrea by specifying:
      
	./configure enable-debugging

      then this parameter causes the actual production of debug
      output. See debug.h for an explication of debugging codes.



******************************
*** The Configuration file ***
******************************

      This section describes the configuration file.

      The configuration file contains directives that alter pgm
      behaviour.

----------------------
-- Some definitions --
----------------------

      First, some definitions.

      * "Excluded" IPs are those that you DON'T want LaBrea to ever
	capture.

        Please note that you don't need to specify "active" IPs (ie
	those with a live machine sitting on the address).  LaBrea
	won't capture an IP with a machine on it. This is only for
	empty IPs that you DON'T want captured.

      * "HardExcluded" IPs are those that you don't want LaBrea to
        hard capture. This is only necessary with the -h option.

---------------------------
-- Specifying directives --
---------------------------

      * IP addresses can be specified as either a single address:
      
	  192.168.33.99

	or as a range of addresses:

	 192.168.0.1 - 192.168.0.50

      * The same thing holds for ports and ranges of ports:

	  22
	  33-55


      * The configuration file consists of lines with two parts: An IP
      or Port (or and IP range or Port range) followed by a "tag". For
      example:

	 192.168.0.1-192.168.0.50 exclude
      

      Blank lines are ignored as are lines starting with "#".

$Id: README,v 3.2 2021/10/10 10:23:39 lorgor Exp $

