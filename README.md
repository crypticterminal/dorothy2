# Dorothy2

A malware/botnet analysis framework written in Ruby.

For a perfect view of this document (images and links), open it through the project's [code repository](https://github.com/m4rco-/dorothy2/blob/master/README.md).

##Introduction

Dorothy2 is a framework created for mass malware analysis. Currently, it is mainly based on analyzing the network behavior of a virtual machine where a suspicious executable was executed.
Additionally, it is able to recognize new spawned processes by comparing them with a previously created baseline.
Static binary analysis and an improved system behavior analysis will be shortly introduced in the next versions.

Dorothy2 is a continuation of my Bachelor degree's final project ([Dorothy: inside the Storm](https://www.honeynet.it/wp-content/uploads/Dorothy/The_Dorothy_Project.pdf) ) that I presented on Feb 2009.
The main framework's structure remained almost the same, and it has been fully detailed in my degree's final project or in this short [paper](http://www.honeynet.it/wp-content/uploads/Dorothy/EC2ND-Dorothy.pdf). More information about the whole project can be found on the Italian Honeyproject [website](http://www.honeynet.it).


The framework is mainly composed by four big elements that can be even executed separately:

* The Dorothy analysis engine (included in this gem)

 In charge of executing a binary file into a sandbox, and then storing the generated network traffic and its screenshots into the analysis folder (moreover populating Dorothive with the basic information of the file, and CouchDB with the network pcaps).

* The (network) Data Extraction Module aka dparser (included in this gem)

 In charge of dissecting the pcaps file, and storing the most relevant information (flows data, GeoIP info, etc) into Dorothive. In addition, it extracts all the files downloaded by the sandbox through HTTP/HTTPS and store them into the binary file's analysis folder.

* The Webgui (Coded in Rails by Andrea Valerio, and not yet included in this gem)

 (Working in progress) A rails application which allows to have an interactive overview on all the acquired data. For a first glance on Andrea's first PoC take a look [this](http://youtu.be/W4DdMYPp4Ws) video (commented in Italian).

* The Java Dorothy Drone (Mainly coded by Patrizia Martemucci and Domenico Chiarito, but not part of this gem and not publicly available.)

 Our botnet infiltration module, refers to this [ppt](https://www.honeynet.it/wp-content/uploads/Presentations/JDrone.pptx) presentation for an overview.

The first three modules are (or will be soon) publicly released under GPL 3 license as tribute to the the [Honeynet Project Alliance](http://www.honeynet.org).
All the information generated by the framework - i.e. binary info, timestamps, dissected network analysis - are stored into a postgres DB (Dorothive) in order to be used for further analysis.
A no-SQL database (CouchDB) is also used to mass store all the traffic dumps thanks to the [pcapr/xtractr](https://code.google.com/p/pcapr/wiki/Xtractr) technology.

I started to code this project in late 2009 while learning Ruby at the same time. Since then, I´ve been changing/improving it as long as my Ruby coding skills were improving. Because of that, you may find some parts of code not-really-tidy :)

##Requirements

>WARNING:
The current version of Dorothy only utilizes VMWare ESX5 as its Virtual Sandbox Module (VSM). Thus, the free version of ESXi is not supported due to its limitations in using the
vSphere 5 API.
However, the overall framework could be easily customized in order to use another virtualization engine. Dorothy2 is
very [modular](http://www.honeynet.it/wp-content/uploads/The_big_picture.pdf),and any customization or modification is very welcome.

Dorothy needs the following software (not expressly in the same host) in order to be executed:

* VMWare ESX >= 5.0  (tip: if you download ESXi, you can evaluate ESX for 30 days)
* Ruby 1.8.7
* Postgres >= 9.0
* At least one Windows virtual machine
* One unix-like machine dedicated to the Network Analysis Engine(NAM) (tcpdump/ssh needed)
* [pcapr-local](https://github.com/mudynamics/pcapr-local )  (only used by doroParser)
* MaxMind libraries (only used by doroParser)

Regarding the Operating System

* Dorothy has been designed to run on any *nix system. So far it was successfully tested on OSX and Linux.
* The virtual machines used as sandboxes are meant to be Windows based (successfully tested on XP)
* Only pcapr-local strictly requires Linux, if you want to use a Mac for executing this gem (like I do), install it into the NAM (as this guide suggests)

## Installation

It is recommended to follow this step2step process:

1. Set your ESX environment
  * Sample setup
2. Install the required software
3. Install Dorothy and libmagic libraries
4. Start Dorothy, and configure it
5. Use Dorothy

### 1. Set your ESX environment
1. Basic configuration (ssh)
 * From vSphere:

            Configuration->Security Profile->Services->Proprieties->SSH->Options->Start and Stop with host->Start->OK

2. Configure two separate virtual networks, one dedicated exclusively to the SandBoxes (See Sample Setups)

3. Configure the Windows VMs used for sandboxing

 * Disable Windows firewall (preferred)
 * VMWare Tools must be installed in the Windows guest system.
 * Configure a static IP
 * After configuring everything on the Guest OS, create a snapshot of the sandbox VM from vSphere console. Dorothy will use it when reverting the VM after a binary execution.

3. From vSphere, create a unix VM dedicated to the NAM


* Install tcpdump and sudo

        #apt-get install tcpdump sudo

* Create a dedicated user for dorothy (e.g. "dorothy")

        #useradd dorothy

* Create a directory inside the dorothy user's home where storing the network dumps

        #su dorothy
        $mkdir /home/dorothy/pcaps

* Add dorothy's user permission to execute/kill tcpdump to the sudoers file:

        #visudo
        add the following line:
        dorothy  ALL = NOPASSWD: /usr/sbin/tcpdump, /bin/kill, /usr/bin/killall

* If you want to install pcapr on this machine (if you want to use dorohy from a MacOSX machine, you have to do it) install also these packages  (refer to this blog [post](https://github.com/pcapr-local/pcapr-local) for a detailed howto). However, if you are installing Dorothy into a Linux machine, I recommended you to install pcapr on the same machine where the Dorothy gem was installed.

        #apt-get install ruby1.8 rubygems  tshark zip couchdb

* Start the couchdb server

        #/etc/init.d/couchdb start

* Install pcapr-local

        #gem install pcapr-local

* Start pcapr-local by using the dorothy's system account and configure it. When prompted, insert the folder path used to store the network dumps

        $startpcapr
        ....
        Which directory would you like to scan for indexable pcaps? [/root/pcapr.Local/pcaps]
        /home/dorothy/pcaps

 In addition, remember to allow pcapr to run on all the interfaces

        What IP address should pcapr.Local run on? Use 0.0.0.0 to listen on all interfaces [127.0.0.1]
        0.0.0.0

* If everything went fine, you should be able to browse to

        http//{ip-used-by-NAM}:8000

4 From vSphere, configure the NIC on the virtual machine that will be used for the network sniffing purpose (NAM).
       >The vSwitch where the vNIC resides must allow the promisc mode, to enable it from vSphere:

       >Configuration->Networking->Proprieties on the vistualSwitch used for the analysis->Double click on the virtual network used for the analysis->Securiry->Tick "Promiscuous Mode", then select "Accept" from the list menu.


#### * Sample Setups
1. Basic setup
> In the following example, the Dorothy gem is installed in the same host where Dorothive (the DB) resides.
> This setup is strongly recommended

    >![dorothy.basicsetup](http://www.honeynet.it/wp-content/uploads/Dorothy-Basic.jpg)

2. Advanced setup
> This setup is recommended if Dorothy is going to be installed in a Corporate environment.
> By leveraging a private VPN, all the sandbox traffics exits from the Corporate network with an external IP addresses.

 >![dorothy.advancedsetup](http://www.honeynet.it/wp-content/uploads/Dorothy-Advanced.jpg)

### 2. Install the required software


1. Install postgres

        $sudo apt-get install postgresql-9.1
or

        http://www.postgresql.org/download/

2. Configure a dedicated postgres user for Dorothy (or use the default postgres user instead, up to you :)

> Note:
> If you want to use Postgres "as is", and then configure Dorothy to use "postgres" default the user, configure a password for this user at least (by default it comes with no password)

3. Install the following packages

        $sudo apt-get install ruby1.8 rubygems postgresql-server-dev-9.1 libxml2-dev  libxslt1-dev libmagic-dev

>For OSX users: all the above software are available through mac ports. A tip for libmagic: use brew instead:
>
        $ brew install libmagic
        $ brew link libmagic

In case you want to install pcapr here do this as well:

        $sudo apt-get install tshark zip couchdb

### 3. Install Dorothy gem

*Install Dorothy gem

        $ sudo gem install dorothy2

### 4. Start Dorothy, and configure it!

0. Install MaxMind libraries
    * [GeoLiteCity](http://geolite.maxmind.com/download/geoip/database/GeoLiteCity.dat.gz)
    * [GeoLite ASN](http://download.maxmind.com/download/geoip/database/asnum/GeoIPASNum.dat.gz)
    * Copy GeoLiteCity.dat and GeoIPASNum.dat into Dorothy's etc/geo/ folder

1. Start Dorothy

        $ dorothy_start -v
The following message should appear

        [WARNING] It seems that the Dorothy configuration file is not present,
        please answer to the following question in order to create it now.

2. Follow the instruction to configure
    * The environment variables (db, esx server, etc)
    * The Dorothy sources (where to get new binaries)
    * The ESX Virtual machines used for the analysis

The first time you execute Dorothy, it will ask you to fill those information in order to create the required configuration files into the etc/ folder. However, you are free to modify/create such files directly - configuration example files can be found there too.
Finally, check out the file extensions.yml within the /etc folder: it instructs Dorothy's sandboxes about how to process the binaries to analize.

###5. Use Dorothy
1. Copy a .exe or .bat file into $yourdorothyhome/opt/bins/manual/
2. Execute dorothy with the malwarefolder source type (if you left the default one)

    $ dorothy_start -v -s malwarefolder


## Usage

### Dorothy usage:

	$dorothy_start [options]
	where [options] are:
       --Verbose, -V:   Print the current version
       --verbose, -v:   Enable verbose mode
      --infoflow, -i:   Print the analysis flow
      --baseline, -b:   Create a new process baseline
    --source, -s <s>:   Choose a source (from the ones defined in etc/sources.yml)
        --daemon, -d:   Stay in the background, by constantly pooling datasources
        --manual, -m:   Start everything, copy the file, and wait for me.
  --SandboxUpdate, -S:  Update Dorothive with the new Sandbox file
  --DorothiveInit, -D:  (RE)Install the Dorothy Database (Dorothive)
          --help, -h:   Show this message


 >Example
>
     $dorothy_start -v -s malwarefolder

After the execution, if everything went fine, you will find the analysis output (screens/pcap/bin) into the analysis folder that you have configured e.g. dorothy/opt/analyzed/[:digit:]/
Other information will be stored into Dorothive.
If executed in daemon mode, Dorothy2 will poll the datasources every X seconds (where X is defined by the "dtimeout:" field in the configuration file) looking for new binaries.

### DoroParser usage:

    $dparser_start [options]
	where [options] are:
    --verbose, -v:   Enable verbose mode
    --nonetbios, -n:   Hide Netbios communication
     --daemon, -d:   Stay in the backroud, by constantly pooling datasources
       --help, -h:   Show this message

 >Example
>
     $dparser_start -d start
     $dparser_stop


After the execution, if everything went fine, doroParser will store all the donwloaded files into the binary's analysis folder e.g. dorothy/opt/analyzed/[:digit:]/downloads
Other information -i.e. Network data- will be stored into Dorothive.
If executed in daemon mode, DoroParser will poll the database every X seconds (where X is defined by the "dtimeout:" field in the configuration file) looking for new pcaps that has been inserted.

###6. Debugging problems

I recognize that setting up Dorothy is not the easiest task of the world.
By considering that the whole framework consists in the union of several 3rd pats, it is very likely that one of them will fail during the process.
Below there are some tips about how understand the root-cause of your crash.

1. Execute the Dorothy UnitTest (tc_dorothy_full.rb) that resides in its gem home directory

 >Example

    $cd /opt/local/lib/ruby/gems/1.8/gems/dorothy2-0.0.1/test/
    $ruby tc_dorothy_full.rb

2. Set the verbose flag (-v) while executing dorothy

> $dorothy_start -v -s malwarefolder

3. If any error occours, go to our Redmine and raise a [bug-ticket](https://redmine.honeynet.it/projects/dorothy2/)!    

------------------------------------------

## Acknowledgements

Thanks to all the people who have contributed in making the Dorothy2 project up&running:

* Marco C. (research)
* Davide C. (Dorothive)
* Andrea V. (WGUI)
* Domenico C. - Patrizia P. (Dorothive/JDrone)
* [All](https://www.honeynet.it/research) the graduating students from [UniMI](http://cdlonline.di.unimi.it/) who have contributed.
* Sabrina P. (our students "headhunter" :)
* Jorge C. and Nelson M. (betatesting/first release feedbacks)

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request

Every contribution is more than welcome!
For any help, please don't hesitate in contacting us at :
info at honeynet.it

## License

Dorothy is copyrighted by Marco Riccardi and is licensed under the
following GNU General Public License version 3.

                    GNU GENERAL PUBLIC LICENSE
                       Version 3, 29 June 2007

 Copyright (C) 2007 Free Software Foundation, Inc. <http://fsf.org/>
 Everyone is permitted to copy and distribute verbatim copies
 of this license document, but changing it is not allowed.
