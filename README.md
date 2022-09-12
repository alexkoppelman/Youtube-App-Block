# Youtube-App-Block
Use pihole groups to block access to youtube apps on mobile/alexa/... on your local network

I use a Pi 4 for the pihole install. This is connected to the router by cable. Following the pihole install, you can also use the pi as a wireless access point, but in my case I have enough wireless stuff going on;)

Steps
1. Install PiHole
2. Create Group(s)
3. Create Clients
4. Add Domains
5. Create script (linux)

Optional:

6a. Run script on timer using cron and/or
6b. Create a flow in node-red to do the switching for you


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
![image](https://user-images.githubusercontent.com/14348439/189656578-4a2eab1b-dac5-4cb6-8c77-329a23608605.png)


# 5. Create script to switch group On or Off
If group is on, devices in the group have youtube blocked. If the group is off, the devices in the group are set to the "Default" group with no blocking.
You can do this through the pihole web front-end, but why not use a script:

**sudo nano yourscriptname.sh**
Paste the below script and update with your details (make sure to comment out the stuff you don't need, logfile? mongodb update?)
Use sqlite to get the group ID of your group: **sqlite3 /etc/pihole/gravity.db "select * from 'group' ;"** (the first number in the outbut is your group ID
Save and close
Make executable (**chmod +x yourscriptname.sh**)

Now run your script (**./yourscriptname.sh**) and check if it all worked.

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

# 6a Run script on timer using cron
Use this [tutorial](https://www.cyberciti.biz/faq/how-do-i-add-jobs-to-cron-under-linux-or-unix-oses/) to schedule the switching script.

# 6b. Create a flow in node-red to do the switching for you

Using the dashboard flows, you can easliy create a button and feedback text. I had a small added difficulty as my node-red instance runs on a different pi than the pihole instance. This flow kicks off a script that runs the switching script through an ssh command.

```
[{"id":"1ca42ec55694aa29","type":"ui_button","z":"09c0558fa6e0cecd","name":"Switch YT","group":"d9eb2021.565ca8","order":3,"width":0,"height":0,"passthru":false,"label":"--SWITCH--","tooltip":"","color":"","bgcolor":"","className":"","icon":"","payload":"","payloadType":"str","topic":"topic","topicType":"msg","x":160,"y":580,"wires":[["3b3932c5e9cfe466"]]},{"id":"3b3932c5e9cfe466","type":"exec","z":"09c0558fa6e0cecd","command":"/home/pi/toggleYTremote.sh","addpay":"","append":"","useSpawn":"false","timer":"","winHide":false,"oldrc":false,"name":"","x":380,"y":580,"wires":[["358d46a22f0ddeb3"],[],[]]},{"id":"2616311a6e61660b","type":"ui_text","z":"09c0558fa6e0cecd","group":"d9eb2021.565ca8","order":1,"width":0,"height":0,"name":"","label":"Youtube is now","format":"{{msg.payload}}","layout":"row-left","className":"","x":840,"y":580,"wires":[]},{"id":"358d46a22f0ddeb3","type":"function","z":"09c0558fa6e0cecd","name":"","func":"var onoff = msg.payload.slice(-2);\nonoff = Array.from(onoff)[0];\n\nif(onoff == 1) {\n    msg.payload = \"On\"\n} else {\n    msg.payload = \"Off\"\n}\n\n\n\n\nreturn msg;","outputs":1,"noerr":0,"initialize":"","finalize":"","libs":[],"x":620,"y":580,"wires":[["2616311a6e61660b"]]},{"id":"d9eb2021.565ca8","type":"ui_group","name":"YT Control","tab":"208bb4d618807bb5","order":1,"disp":true,"width":"6","collapse":true,"className":""},{"id":"208bb4d618807bb5","type":"ui_tab","name":"YT Control","icon":"dashboard","disabled":false,"hidden":false}]
```
