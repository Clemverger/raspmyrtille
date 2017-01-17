If you don’t already have a subfolder inside your user’s home folder for scripts, create one now:

mkdir ~/bin
Now copy this script into your favourite text editor, and save it as network-monitor.sh inside that folder.

#!/bin/bash

LOGFILE=/home/admin/network-monitor.log

if ifconfig eth0 | grep -q "inet addr:" ;
then
        echo "$(date "+%m %d %Y %T") : Ethernet OK" >> $LOGFILE
else
        echo "$(date "+%m %d %Y %T") : Ethernet connection down! Attempting reconnection." >> $LOGFILE
        ifup --force eth0
        OUT=$? #save exit status of last command to decide what to do next
        if [ $OUT -eq 0 ] ; then
                STATE=$(ifconfig eth0 | grep "inet addr:")
                echo "$(date "+%m %d %Y %T") : Network connection reset. Current state is" $STATE >> $LOGFILE
        else
                echo "$(date "+%m %d %Y %T") : Failed to reset ethernet connection" >> $LOGFILE
        fi
fi
i.e.

nano ~/bin/network-monitor.sh
…then copy & paste, save and exit (Ctrl + X, hit yes when prompted to save). Make sure the long lines don’t get truncated when you copy and paste them over, or the script won’t work!

Note that the script points to a file that will be used as a log on line 3. My username is “admin”; if yours is “pi” then change that line to LOGFILE=/home/pi/network-monitor.log, or replace “pi” with any other username.

Adding ~/bin to your $PATH variable, and making the script executable

Your $PATH variable is a list of places that the shell looks for executables. We need to add the newly added ~/bin so that you don't have to use the full path to the script when you run it. To do this, open ~/.bashrc and add this line to the end:

PATH=$PATH:~/bin
The shell normally reads ~/.bashrc when you log in, but we can tell it to parse it now using this command to that the changes take effect:

source ~/.bashrc
Now we need to make the script executable, e.g.:

chmod +x ~/bin/network-monitor.sh
Initial test

You should now be able to run the script using sudo (i.e. sudo network-monitor.sh). Check the log file with this command:

less ~/network-monitor.log
You should see a date and time stamp from when you ran the script, followed by “Ethernet OK”. If this is what you see, all is well. Press q to quit “less” and return to the command prompt when you are done.

Automating with cron

Cron is a tool that can run scripts at regular intervals, and is very suited for this kind of thing. Luckily, it’s really easy to get started with.

Open the cron configuration file with root privileges:

sudo nano /etc/crontab
Now schedule the script to be run every 5 minutes (or any other interval that you would prefer), by adding this last line to the end of the file (remember to change “admin” if you have a different username):

# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.d$
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.w$
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.m$
#

*/5 * * * * root bash /home/admin/bin/network-monitor.sh
If you want the script to run every 10 minutes (or any other multiple of 1 minute), change the first part of the line, e.g. to “*/10″. It’s important to set the script to run with root because the ifup command requires superuser privileges.

Make a cup of tea and come back. Check the log file again, you should see more entries with time stamps that are 5 minutes (or whatever you specified) apart.

Full Test

Now is the time to test your script to see if it will reconnect you when the ethernet connection goes down.

I’d recommend that you have physical access to the Pi when you test this for the first time, just in case you have to pull the power and reboot (very unlikely if everything has worked so far).

Make sure that your Cron interval isn’t set too high or you’ll have to wait ages (5 minutes is about right for testing).

Connect to the Pi via SSH, and issue this command (after you issue it, your SSH session will become unresponsive, but that’s kind of the point!):

sudo ifdown eth0
Wait for at least the amount of time you specified between cron jobs, and then login again. You may actually still be logged in – if you didn’t try and get the Pi to do anything while the ethernet was down, it may not have registered the connection drop.

Check the log file. You should see something like this:
