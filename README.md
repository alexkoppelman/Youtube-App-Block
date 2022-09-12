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
make sure you setup your pihole server as the main DNS resolver in your router config.

# 2. Create goup
Create a group for the devices that you'll want to block. Mine's called: youtubeOnOff
![image](https://user-images.githubusercontent.com/14348439/189654135-a2a92bba-8a39-4738-88d8-46ebec3a4b19.png)

# 3. Create Clients
Make sure to add all devices using youtube to the group created for this earlier
![image](https://user-images.githubusercontent.com/14348439/189654665-c396a904-afa5-4e2b-b893-0378c408f0ca.png)

# 4. Add Domains:  (regex blacklist)
```
(\.|^)googlevideo\.com$
(\.|^)youtubei\.googleapis\.com$
(\.|^)youtube\.com$
```

# 5. Create script to switch group on Off
If group is on, devices in the group have youtube blocked. If the group is off, the devices in the group are set to the "Default" group with no blocking.
You can do this through the pihole web front-end, but why not use a script:

**sudo nano yourscriptname.sh**
Paste the below script and update with your details (make sure to comment out the stuff you don't need, logfile? mongodb update?)
use sqlite to get the group ID of your group: **sqlite3 /etc/pihole/gravity.db "select * from 'group' ;"** (the first number in the outbut is your group ID
Save and close
make executable (**chmod +x yourscriptname.sh**)



```
  #!/bin/bash

  #get current group setting - Update with your GroupID
  enabled=$(sqlite3 /etc/pihole/gravity.db "select enabled from 'group' WHERE id = '3';")

  #switch to new setting
  ((enabled ^= 1))

  #make timestamp
  TIMESTAMP=$(date "+%Y-%m-%d %H:%M:%S")

  # change Pihole group setting  - Update with your GroupID
  sudo sqlite3 /etc/pihole/gravity.db "update 'group' set enabled = $enabled WHERE id = '3';"

  # add line to logfile - If needed?
  echo "YT BlockGroup ${enabled} - ${TIMESTAMP} " >> /home/pi/scripts/toggleYT.log

  # add row to mongodb for logging purposes - If needed?
  mongo -u admin -p ************ --authenticationDatabase admin 127.0.0.1/IOTdatabase --eval "var document = { status : \"$enabled\", datetime : \"$TIMESTAMP\" }; db.youtubeBlock.insert(document);"

  #Restart relevant pihole services
  pihole restartdns reload-lists

```
