# avahi-0.8-rtlsdr

# Purpose: 

To build Avahi for Openwrt with an entry added to support an rtl_tcp server.

# Prerequisite's/Environment

This solution was created on Ubuntu Linux 22 LTS, 
and creates the packages for TP-Link's Archer C7 v2,
running OpenWRT v23.05.3. It has been successfully tested for gl-ar300m wth v23.05.3 also.   

it has not been tested or tried on any other OSes or routers  

See https://openwrt.org/docs/guide-developer/toolchain/install-buildsystem for other OS setups.

And, in general, most of what is done here comes directly from the OpenWRT Developer's guide tutorial,

found here: https://openwrt.org/docs/guide-developer/helloworld/chapter1 

# Process

On Ubuntu, run the following to install necessary OS tools:

sudo apt update
sudo apt install build-essential clang flex bison g++ gawk \
gcc-multilib g++-multilib gettext git libncurses5-dev libssl-dev \
python3-setuptools rsync swig unzip zlib1g-dev file wget

Install OpenWRT's build system source code in a folder of your choosing, the folder names here match those from the tutorial.

mkdir -p /home/buildbot
cd home/buildbot
git clone https://git.openwrt.org/openwrt/openwrt.git source

cd /home/builbot/source
git checkout v23.05.3
make distclean

Update the package feeds:

./scripts/feeds update -a
./scripts/feeds install -a

Configure your build with the configuration UI:

make menuconfig

This is where you tell the build system which chip, flashtype, and board the packages should be built for.

In my case this was Ath79, Generic, TP-Link Archer C7 v2

Save the .config and exit the UI.

Build the toolchain (this takes awhile, tens of minutes or longer on a 4-core i7 laptop from 2017):

make toolchain/install

when finished add the following to your PATH so any needed scripts can be easily run.

export PATH=/home/buildbot/source/staging_dir/host/bin:$PATH


Now we deviate a little bit from the Tutorial. Open the configuration UI again:

make menuconfig

Keep the previous settings for target cpu/board but navigate down the list to the 'Libraries' section.
Scroll down until you see 3 items starting with 'libavahi', these should be 'client', 'dbus-support', and 'no-dbus-support'.
Press Y on each to select them. Save the config and exit to the top level menu.
Now scroll down and enter 'Network' and then 'IP Addresses and Names'. There should be 7 entries starting with 'avahi', press Y on each to select them. Save the config and exit the UI.

Download packages from source feeds:

OpenWRT's build system uses feeds to supply packages with their source code. It is possible to create your own local feeds, but in this case it will pull the official Avahi source for us.

Running the following downloads any needed sources and the source for any packages we have selected in menuconfig:

make download

Once complete, there should be a tar file in /source/dl named 'avahi-0.8.tar.gz'

We are going to modify this source code so that rtl_tcp is added to the service types database that avahi will look for when running.

Open avahi-0.8.tar.gz with Archive Manager and double click the avahi-0.8 folder, then the service-type-database folder and then the 'service-types' file. 

at the bottom of the file after the entry for '_device-info' we'll add an entry for rtl_tcp, I chose:

_rtltcp-server._tcp:RtlTcp Server (SDR)

You can customize this somewhat, but beware of the convention used by Avahi, for more on this see:

https://github.com/avahi/avahi/blob/master/service-type-database/service-types

http://www.dns-sd.org/ServiceTypes.html

and

https://deepwiki.com/avahi/avahi/6-service-type-database


Once the new entry is added, save it, you should be prompted to update the archive, select 'update' and close the archive manager.

It's almost time to compile the package but before we do we have to modify the package's Makefile. The included Makefile will attempt to re-download the source, before compiling thus wiping out the changes to the service types database.

navigate to /source/feeds/packages/libs/avahi and open Makefile for editing. Delete the entry in front of PKG_SOURCE_URL: and enter the full path to the folder your /source folder resides in. Mine looks like:

PKG_SOURCE_URL:/home/USER_NAME/Desktop/buildbot

Save the file.

Compiling Avahi:

Now, simply run make /package/avahi/compile

when finished, navigate to /source/bin/packages/YOUR_BUILD_TARGET/packages

and you should see the package names that we selected earlier in the configuration UI.

Install on OpenWRT:

To install the newly created packages on a router it should have ssh enabled. 

In my case I am using the 'no-dbus' version of libavahi and avahi-daemon, so I copied these 2 ipkg's to a new folder along with the ssh and http service files, and then used scp to upload them to the router, modify these steps as suits your environment:

scp -r file_folder user@routerIP:/

Then ssh'd into the router:

ssh user@routerIP

If you have another version of Avahi installed already you'll have to remove it:

opkg update

opkg list-installed | grep "avahi"

look for any package name with avahi in it and issue

opkg remove [package(s)]

enter the directory that you uploaded the ipkg's into and install the packges:

NOTE: opkg install must be done in this order, if dependencies are not already installed opkg will look to download them from its sources which will not have the changes we made.

opkg install libavahi-dbus_choice-support-.... use <TAB> for completion if necessary

opkg install avahi-dbus-choice-daemon...

opkg install avahi-daemon-service-ssh/http (order doesn't matter with these)

Next we have to create a service file for rtl_tcp.

cd

cd /etc/avahi/services

cp ssh.service rtltcp-server.service

vi rtltcp-server.service

replace <type>_ssh._tcp</type> with <type>_rtltcp-server._tcp</type>

and <port>22</port> with <port>1234</port> OR, whatever port number you choose for your server, 1234 is default

It should be good to go but we can check with:

service avahi-daemon status

if it says 'running' you're all set. 

### TODO ###

Firewall config:

To allow Avahi through the OpenWRT firewall,


Test on a device:

avahi-browse -arp
