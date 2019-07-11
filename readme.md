# On Demand Minecraft Server
Using a Python Flask application and AWS, this repository launches an AWS EC2 Instance to host a Minecraft server upon request from users through the web application. The server will automatically shut down after the server has crashed or is empty for 15 minutes. This makes server hosting for small communities very inexpensive. For up to 20 players you can expect $0.02 per hour the server runs. The largest benefit of this system is that the server bill diminishes if your community decides to take a break from the game, and will be ready to pick back up when you want to play again. No subscriptions are required.

Note that this configuration will likely require familiarity with programming, SSH, and the command line.


# AWS Setup
This step will properly configure your AWS account and configuration.py file so that an instance can be created via the createInstance.py script.

 1. Create or access an **AWS Account**. Under the **User Dropdown** in the    **Toolbar**, select **Security Credentials**, then **Access Keys**, and finally **Create New Access Key**. Download this file, open it, and copy the values of **AWSAccessKeyId** and **AWSSecretKey** to **ACCESS_KEY** and **SECRET_KEY** in the **configuration.py** file in the root directory of the repository.
	
	<code>ACCESS_KEY = 'YourAWSAccessKeyIdHere'
	SECRET_KEY  =  'YourAWSSecretKeyHere'</code> 

 3. Navigate to the **EC2 Dashboard** under the **Services Dropdown** and select **Security Groups** in the sidebar. Select **Create Security Group**, input **minecraft** for the **Security group name**. Create **Inbound Rules** for the following:
	 - Type: **SSH** Protocol: **TCP** Port Range: **22** Source: **Anywhere**
	 - Type: **Custom TCP Rule** Protocol: **TCP** Port Range: **25565** Source: **Anywhere**
	 - Type: **Custom UDP Rule** Protocol: **UDP** Port Range: **25565** Source: **Anywhere**
	 
	 In **configuration.py** in the root directory, set **ec2_secgroups** to the name of the security group, not the id.
	 
	 <code>ec2_secgroups =  ['YourGroupNameHere']</code>

3. Under the **EC2 Dashboard** navigate to **Key Pairs** in the sidebar. Select **Create Key Pair**, provide a name and create. Move the file that is downloaded into the root directory of the project. In **configuration.py** in the root directory, set ** ec2_keypair** to the name entered, and **SSH_KEY_FILE_NAME** to the name.pem of the file downloaded.

	THIS MIGHT BE SUBJECT TO CHANGE
		<code>ec2_keypair =  'YourKeyPairName'
		SSH_KEY_FILE_PATH  =  './YourKeyFileName.pem'</code>

4. This step is concerned with creating the AWS instance. View [https://docs.aws.amazon.com/general/latest/gr/rande.html](https://docs.aws.amazon.com/general/latest/gr/rande.html) (Or google AWS Regions), and copy the  **Region** column for the **Region Name** of where you wish to host your server. In **configuration.py** of the root directory, set the **ec2_region** variable to the copied value.

	<code>ec2_region =  "Your-Region-Here"</code>

5. Navigate to [https://aws.amazon.com/ec2/instance-types/](https://aws.amazon.com/ec2/instance-types/) and select one of the T3 types (with the memory and CPU you desire, I recommend 10 players/GB). Copy the value in the **Model** column. I've configured mine to use **t3.small**. In **configuration.py** of the root directory, set the **ec2_instancetype** variable to the copied value.

	<code>ec2_instancetype =  't3.yourSizeHere'</code>

6. Then we must select an image for the instance to boot. We'll use Alpine Linux becuase it's lightweight and boots fast. Select an image from [https://github.com/mcrute/alpine-ec2-ami](https://github.com/mcrute/alpine-ec2-ami) and copy the **AMI-ID** for your chosen region. In **configuration.py** of the root directory, set the **ec2_amis** variable to the copied value.

	<code>ec2_amis =  ['ami-YourImageIdHere']</code>

7. At this point you should have the necessary configuration to create a new instance through the **createInstance.py** script in the **root** folder. Open a command line in the utilityScripts directory of the project, and execute:

	<code>pip install -r requirements.txt</code>
	
	After successful installation of dependencies execute:

	<code>python utilityScripts/createInstance.py</code>

	Copy the **Instance ID** that is output into the terminal. In **configuration.py** of the root directory, set the **INSTANCE_ID** variable to the copied value.

	<code>INSTANCE_ID  =  'i-yourInstanceIdHere'</code>


# Web Application Deployment
In this step the project will get deployed to Heroku's free hosting. This part of the application provides a rudimentary UI and Web URL for users to start the server.

 1. Create or have access to a Heroku account.
 2. Install and setup the **Heroku CLI** onto your computer. [https://devcenter.heroku.com/articles/heroku-cli#download-and-install](https://devcenter.heroku.com/articles/heroku-cli#download-and-install)
 3. In the command line for the directory of this project, type:
	 <code>heroku create YourProjectNameHere</code>
4. Once this new project has been created, it is time to push the project to Heroku.
	<code>git push heroku master</code>
5. Set the following environmental variables in Heroku: `SERVER_PASSWORD`, `ACCESS_KEY`, `SECRET_KEY`, `EC2_REGION`, `INSTANCE_ID`
6. Acess the site at YourProjectNameHere.herokuapp.com

# AWS Instance Configuration
This step will configure the AWS Linux server to run the minecraft server.
1. SSH into the instance with

	`ssh -i pathToYourKeyFileHere alpine@IPAddress`

2. Install packages with

	<code>sudo apk add wget vim screen openjdk8</code>

3.  Download the desired Minecraft server version from [https://www.minecraft.net/en-us/download/server/](https://www.minecraft.net/en-us/download/server/) into the home directory `/home/alpine`.

4. For some reason `/etc/periodic` doesn't seem to work so we have to manually run the periodic shutdown script. This script checks every 10 minutes if the server is empty and if it is, shuts down the server. Create `autoshutdown.sh` with execute permission:
```
#!/bin/sh
SERVICE='server.jar'
while true; do
  sleep 10m
  if ps ax | grep -v grep | grep $SERVICE > /dev/null; then
    PLAYERSEMPTY=" There are 0 of a max 20 players online"
    $(screen -S minecraft -p 0 -X stuff "list^M")
    sleep 5
    $(screen -S minecraft -p 0 -X stuff "list^M")
    sleep 5
    PLAYERSLIST=$(tail -n 1 /home/alpine/logs/latest.log | cut -f2 -d"/" | cut -f2 -d":")
    echo $PLAYERSLIST
    if [ "$PLAYERSLIST" = "$PLAYERSEMPTY" ]
    then
      echo "Waiting for players to come back in 30s, otherwise shutdown"
      sleep 30
      $(screen -S minecraft -p 0 -X stuff "list^M")
      sleep 5
      $(screen -S minecraft -p 0 -X stuff "list^M")
      sleep 5
      PLAYERSLIST=$(tail -n 1 /home/alpine/logs/latest.log | cut -f2 -d"/" | cut -f2 -d":")
      if [ "$PLAYERSLIST" = "$PLAYERSEMPTY" ]
      then
        echo "powering off"
        $(sudo /sbin/poweroff -d 30)
      fi
    fi
  else
    echo "Screen does not exist, briefly waiting before trying again"
    sleep 1m
    if ! ps ax | grep -v grep | grep $SERVICE > /dev/null; then
      echo "Screen does not exist, shutting down"
      $(sudo /sbin/poweroff -d 30)
    fi
  fi
done
```

4. Create `run.sh` with execute permissions:
```
#!/bin/sh
set -e

SERVER_HOME="/home/alpine"
HEAP_SIZE="2048M"

cd $SERVER_HOME
echo eula=true > eula.txt

case "$1" in
"stop")
  screen -X -S minecraft quit
  ;;
"start")
  /home/alpine/autoshutdown.sh &
  screen -S minecraft -d -m java -server -Xmx${HEAP_SIZE} -Xms${HEAP_SIZE} -jar server.jar nogui
  ;;
esac
```

5. Create `/etc/init.d/minecraft` as root with execute permissions, then run `sudo rc-update add minecraft default`
```
#!/sbin/openrc-run

start() {
  su - alpine -c "/home/alpine/run.sh start"
}

stop() {
  su - alpine -c "/home/alpine/run.sh stop"
}

depend() {
  need net localmount
  after bootmisc
  after firewall
}
```

6. Configure the server as needed, then shutdown with `sudo /sbin/poweroff`

To start the server, access the heroku app, type in the password, and hit start server. The server should take at most a minute and a half to start because the minecraft server takes time to load the spawn area.

# Additional Remarks
## UI Configuration
The title and header for the site can be changed in **/templates/index.html**. Feel free to add any more content or styling to the site, though be careful not to change any form, input, button, or script elements in the template.

## Server Maintenance
Maintaining the server is fairly straightforward. Updating the server file can be done by downloading the new server file, renaming it to **server.jar** and replacing the old file on the server. The world file can be backed up to your PC manually though there is no automated process at this time.

# Credits
forked from https://github.com/trevor-laher/OnDemandMinecraft

alpine linux scripts taken from https://github.com/csauve/servercraft
