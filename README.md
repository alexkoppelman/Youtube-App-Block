# Youtube-App-Block
Use pihole groups to block access to youtube apps on mobile/alexa/... on your local network

I use a respberry for the pihole install. This is connected to the router by cable. Following he pihole install, you can also use the pi as a wireless access point, but in my case I have enough wireless stuff going on;)

Steps
1. Install PiHole
2. Create Group(s)
3. Create Clients
4. Add Domains
5. Create script (linux)
Optional:
6a. run script on timer using cron and/or
6b. create a flow in node-red to do the switching for you


# 1. Install PiHole
Follow the steps outlined here: [Pihole docs](https://docs.pi-hole.net/main/basic-install/)

Create goup:
	youtubeOnOff

Create Clients:
	make sure to add all devices using youtu


add domains:  (regex blacklist)
	(\.|^)googlevideo\.com$
	(\.|^)youtubei\.googleapis\.com$
	(\.|^)youtube\.com$


create script to swicth group on Off
if group is on, devices in the group have youtube blocked. If the roup is off, the devices in the group are set to the "Default" group with no blocking.
	
  #!/bin/bash

  #get current group setting
  enabled=$(sqlite3 /etc/pihole/gravity.db "select enabled from 'group' WHERE id = '3';")

  #switch to new setting
  ((enabled ^= 1))

  #make timestamp
  TIMESTAMP=$(date "+%Y-%m-%d %H:%M:%S")

  # change Pihole group setting 
  sudo sqlite3 /etc/pihole/gravity.db "update 'group' set enabled = $enabled WHERE id = '3';"

  # add line to logfile
  echo "YT BlockGroup ${enabled} - ${TIMESTAMP} " >> /home/pi/scripts/toggleYT.log

  # add row to mongodb for logging purposes
  mongo -u admin -p ************ --authenticationDatabase admin 127.0.0.1/IOTdatabase --eval "var document = { status : \"$enabled\", datetime : \"$TIMESTAMP\" }; db.youtubeBlock.insert(document);"

  #Restart relevant pihole services
  pihole restartdns reload-lists
