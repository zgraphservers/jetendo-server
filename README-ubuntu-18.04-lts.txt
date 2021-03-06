TODO: 
	important, i left monit disabled
	i didn't finish apparmor for new tomcat
	I could use ZFS still in a new version for better disk usage and backup.
	php7.2-fpm is not able to start

Jetendo Server Installation Documentation
OS: Ubuntu Server 18.04 LTS

This readme is for users that want to install Jetendo Server and Jetendo from scratch.
If you downloaded the pre-configured virtual machine from https://www.jetendo.com/ , please use README-FOR-VIRTUAL-MACHINE.txt to get started with it.
If you don't have README-FOR-VIRTUAL-MACHINE.txt, please download it from https://github.com/jetendo/jetendo-server/ 

Download Jetendo Server 
	If you're reading this readme and haven't cloned or downloaded a release of the Jetendo Server project to your host system, please do so now.
	
	You can grab the latest development version from https://github.com/jetendo/jetendo-server/ or a release version from https://www.jetendo.com/
	
	The Jetendo Server project holds most of the configuration required by the virtual machine.

Virtualbox initial setup
	Ubuntu Linux x64
	Minimum requirements: 2048mb ram, 5gb hard drive, 1gb 2nd hard drive for swap, 1 NAT network adapter
	NAT Advanced Settings -> Port forwarding
		Name: SSH, Host Ip: 127.0.0.2: Host Port: 22, Guest Ip: 10.0.2.15, Guest Port: 22
	Setup Shared Folders - The following names must point to the directory with the same name on your host system.  By default, they are a subdirectory of this README.txt file, however, you may relocate the paths if you wish.
		nginx
		mysql
		coldfusion
		system
		lucee
		php
		apache
		jetendo
	Download and mount Ubuntu 18.04 Server LTS ISO to cdrom on first boot

		make / (root) partition on at least a 3gb drive - 8gb+ recommended.
		setup /var on a large separate drive (20gb+) and one large partition, so that the base file is as small as possible with all variable data on the second drive, which can be cloned and attached to multiple virtual machines.
		use defaults for all options - except don't encrypt your home directory.
	The username created during install will not be used later.
	Don't select any packages when prompted because this is a minimal install and they will be configured with shell afterwards.
	
	After finishing the rest of this guide, you'll be able to access:
		SSH/SFTP with:
			127.0.0.2 port 22
		Apache web sites with:
			www.your-site.com.127.0.0.3.nip.io
		Nginx web sites with:
			www.your-site.com.127.0.0.2.nip.io
		Lucee administrator:
			http://127.0.0.2:8888/lucee/admin/server.cfm
		Jetendo Administrator:
			https://jetendo.your-company.com.127.0.0.2.nip.io/z/server-manager/admin/server-home/index
			
	To run other copies of the virtual machine, just update the IP addresses to be unique.  You can use any ips on 127.x.x.x for this without reconfiguring your host system.
			
After OS is installed:		

# run this to act as root for a while:
sudo -i

# change vi default so insert mode is easier to use.  Type these commands:
	vi /root/.vimrc
	press i key twice.
	set nocompatible
	set backspace=indent,eol,start
	Press escape key
	:wq
	Now vi insert mode is easier to use by just pressing i once.
	
	#add a line to vi /etc/vim/vimrc	
		set background=dark

# disable bash history storage
	vi /root/.profile
		unset HISTFILE
	# and run the command once
	unset HISTFILE
	rm /root/.bash_history
	
# force grub screen to NOT wait forever for keypress on failed boot:
	vi /etc/default/grub
		GRUB_RECORDFAIL_TIMEOUT=2
	
	# force ubuntu to boot after 2 second delay on grub menu
		vi /etc/grub.d/00_header
			# put this below ALL the other "set timeout" records in make_timeout
			set timeout=2
			
	update-grub
	
TODOMISSING: # vi /etc/init/cron.conf
	change "exec cron" to "exec cron -L 0"  to stop it from filling syslog with non-error messages.
	and
#vi /etc/rsyslog.d/50-default.conf
	change 
		*.*;auth,authpriv.none		-/var/log/syslog
	to
		*.*;auth,authpriv.none,cron.none		-/var/log/syslog

# Initial kernel & OS update
	sudo apt-get update
	sudo apt-get upgrade
	sudo apt-get dist-upgrade
	
# Enable empty password autologin for root on development server
	sudo passwd root
	vi /etc/pam.d/common-auth
		change nullok_secure to nullok
	vi /etc/ssh/sshd_config
		PermitEmptyPasswords Yes
	vi /etc/shadow
		delete the password hash for root between the 2 colons so it appears like "root::" on the first line.
	
	service ssh restart

# configure firewall	
	# allow web traffic:
	sudo ufw allow 80/tcp
	sudo ufw allow 443/tcp
	
	# allow ssh from specific ip, replace YOUR_STATIC_IP with a real IP address.
	ufw allow from YOUR_STATIC_IP to any port 22
	
	# don't have a static ip? Then allow from any IP (less secure)
	sudo ufw allow 22/tcp
	
	# disable all firewall logging, unless you have concerns
	ufw logging off
	
	# Add connection limiting on a production server
	
		add to /etc/ufw/before.rules after the "drop INVALID packets" configuration lines
		
# Limit to 30 concurrent connections on port 80 per IP
-A ufw-before-input -p tcp --syn --dport 80 -m connlimit --connlimit-above 30 -j REJECT
-A ufw-before-input -p tcp --syn --dport 443 -m connlimit --connlimit-above 30 -j REJECT

# Limit to 20 connections on port 80 per 1 seconds per IP
-A ufw-before-input -p tcp --dport 80 -i eth0 -m state --state NEW -m recent --set
-A ufw-before-input -p tcp --dport 80 -i eth0 -m state --state NEW -m recent --update --seconds 1 --hitcount 20 -j REJECT
-A ufw-before-input -p tcp --dport 443 -i eth0 -m state --state NEW -m recent --set
-A ufw-before-input -p tcp --dport 443 -i eth0 -m state --state NEW -m recent --update --seconds 1 --hitcount 20 -j REJECT

# jetendo - block invalid tcp commands
-A ufw-before-input -p TCP --tcp-flags ALL FIN,URG,PSH -j DROP
-A ufw-before-input -p TCP --tcp-flags ALL SYN,RST,ACK,FIN,URG -j DROP
-A ufw-before-input -p TCP --tcp-flags SYN,RST SYN,RST -j DROP
-A ufw-before-input -p TCP --tcp-flags SYN,FIN SYN,FIN -j DROP
-A ufw-before-input -p TCP --tcp-flags SYN,ACK NONE -j DROP
-A ufw-before-input -p TCP --tcp-flags RST,FIN RST,FIN -j DROP
-A ufw-before-input -p TCP --tcp-flags SYN,URG SYN,URG -j DROP
-A ufw-before-input -p TCP --tcp-flags ALL SYN,PSH -j DROP
-A ufw-before-input -p TCP --tcp-flags ALL SYN,ACK,PSH -j DROP
	service ufw restart
	
	ufw enable

Log out and login as root using ssh for the rest of the instructions.

# If server is a virtualbox virtual machine
	apt-get install build-essential linux-headers-$(uname -r) dkms
	
	mount /dev/cdrom /media
	sh /media/VBoxLinuxAdditions.run
	umount 
	
	# force the vboxsf dkms kernel module to load before fstab runs
		vi /etc/default/rcS
			# add the following line to the bottom of the file:
			/sbin/modprobe vboxsf
	
	# verify the kernel modules are loaded:
		lsmod | grep vbox
	
			
# update hostname
	for development environment, make sure /etc/hostname matches the value used in the Jetendo configuration for the testDomain affix.  I.e. jetendo.127.0.0.2.nip.io
	
	hostnamectl set-hostname jetendo.test.zsite.info
	change contents of /etc/hostname to jetendo.test.zsite.info
	vi /etc/cloud/cloud.cfg
		change preserve_hostname to true

# If this is a virtual machine: Add the contents of /jetendo-server/system/jetendo-fstab.conf and copy the file to /etc/fstab, then run
	mkdir /var/jetendo-server/
	cd /var/jetendo-server/
	mkdir apache nginx mysql php system lucee coldfusion jetendo backup server config custom-secure-scripts logs virtual-machines luceevhosts
	mount -a
	mount mysql fails until it is installed because user doesn't exist yet.
	
Add Prerequisite Repositories
	sudo add-apt-repository ppa:linuxuprising/java
	NOT NEEDED add-apt-repository ppa:kirillshkrogalev/ffmpeg-next
	add-apt-repository ppa:stebbins/handbrake-releases
	add-apt-repository "deb http://archive.ubuntu.com/ubuntu $(lsb_release -sc) universe"
	
	sudo apt-get install software-properties-common
	sudo apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xF1656F24C74CD1D8
	sudo add-apt-repository 'deb [arch=amd64,arm64,ppc64el] http://nyc2.mirrors.digitalocean.com/mariadb/repo/10.3/ubuntu bionic main' 
 
Install Required Packages
	apt-get update
	apt-get install mariadb-server
	
CONTINUE HERE

	apt-get install apache2  cifs-utils samba libsasl2-modules postfix openjdk-11-jdk imagemagick git libssl-dev build-essential libpcre3-dev unzip apparmor-utils php-pear make dnsutils
	timedatectl set-ntp on
	timedatectl set-timezone America/New_York
	  
	apt-get install php7.2
	apt-get install php7.2-mysql php7.2-cli php7.2-gd php7.2-curl php7.2-dev php7.2-sqlite3 curl
	
	apt-get install php7.2-common php7.2-mbstring php7.2-imap php-imagick php7.2-fpm handbrake-gtk handbrake-cli rng-tools p7zip-full mailutils fail2ban opendkim opendkim-tools dnsmasq ffmpeg monit sshpass handbrake-cli
	
	# if you want to use php with apache2, also run this:
	apt-get install libapache2-mod-php7.2
	
	# accept defaults for all installers - when postfix installer prompts you, i.e. OK, Internet Site
	
Configure MariaDB
	Important Note: Mariadb doesn't appear to work when the datadir is in a Virtualbox Shared Folder anymore.  For virtual machines where you want to separate the datadir, you must use a vdi file instead.

	service mysql stop
	#make sure mysql shared folder is mounted if using virtualbox
		mount -a
	
	# to begin with a fresh database, run this command to overwrite your mysql/data folder. WARNING:  If you existing mysql data files on your host system already, don't run this command.
	mkdir /var/jetendo-server/mysql/data/
	cp -rf /var/lib/mysql/* /var/jetendo-server/mysql/data/
	chown -R mysql:mysql /var/jetendo-server/mysql/data/
	cp /var/jetendo-server/system/jetendo-mysql-development.cnf /etc/mysql/my.cnf
	
	# disable the /root/.mysql_history file
	export MYSQL_HISTFILE=/dev/null
	
	you must get the password in /etc/mysql/debian.cnf, and create "debian-sys-maint" user with host: localhost AND 127.0.0.1 with global access to all privileges for service mysql restart to work correctly.

Configure Apache2 (Note: Jetendo CMS uses Nginx exclusive, Apache configuration is optional)
	# enable modules
		a2enmod ssl rewrite proxy proxy_html xml2enc
	Change apache2 ip binding
		vi /etc/apache2/ports.conf
			ServerName dev
			Listen 127.0.0.3:80
			Listen 127.0.0.3:443
	
	service apache2 restart
	
	If you don't need Apache, it is recommended to disable it from starting with the following command:
		update-rc.d apache2 disable
	To re-enable:
		update-rc.d apache2 enable
		service apache2 start
		
Imagemagick 6.9 is the default, but we might need Imagemagick 7, which requires compilation.  You don't have to uninstall the other imagemagick.  These are the steps to do it:
	add line below to: /etc/apt/sources.list
		deb-src http://archive.ubuntu.com/ubuntu/ bionic main restricted
	apt-get update
	apt-get build-dep imagemagick
	wget https://www.imagemagick.org/download/ImageMagick.tar.gz
	tar xf ImageMagick.tar.gz
	cd ImageMagick-7*

	./configure
	make
	make install
	ldconfig /usr/local/lib
	identify -version
	
	
Install OpenSSL 1.1.1+ for TLS 1.3 and more
	mkdir /root/openssltemp
	cd /root/openssltemp
	wget https://www.openssl.org/source/openssl-1.1.1-pre8.tar.gz
	tar xvf openssl-1.1.1-pre8.tar.gz
	cd openssl-1.1.1-pre8
	./config --prefix=/usr/local/ssl --openssldir=/usr/local/ssl -Wl,-rpath,/usr/local/ssl/lib
	make
	make install
	add openssl path to the BEGINNING of the PATH variable in /etc/environment and save
		/usr/local/ssl/bin
		test with:
			openssl version
	
Install Required Software From Source
	Nginx
		mkdir /root/nginx-build
		cd /root/nginx-build
		wget http://nginx.org/download/nginx-1.15.2.tar.gz
		tar xvfz nginx-1.15.2.tar.gz
		adduser --system --no-create-home --disabled-login --disabled-password --group nginx
		
		Put "sendfile off;" in nginx.conf on test server when using virtualbox shared folders
		
		#download and unzip nginx modules
			cd /root/nginx-build/
			wget https://github.com/simpl/ngx_devel_kit/archive/master.zip
			unzip master.zip -d /root/nginx-build/
			rm master.zip
			wget https://github.com/agentzh/set-misc-nginx-module/archive/master.zip
			unzip master.zip -d /root/nginx-build/
			rm master.zip
			
			wget https://github.com/fdintino/nginx-upload-module/archive/master.zip
			unzip master.zip -d /root/nginx-build/
			rm master.zip
			
			#embed java into nginx
			wget https://github.com/nginx-clojure/nginx-clojure/archive/master.zip
			unzip master.zip -d /root/nginx-build/
			rm master.zip
			
			#need java 8 due to use of UNSAFE
			apt install openjdk-8-jre-headless
			
				# because of new version of gcc, I had to change comments that we /*no break*/ to /*fallthrough*/ in these 2 files to get the warnings to go away for make:
					/root/nginx-build/nginx-clojure-master/src/c/ngx_http_clojure_mem.c
					/root/nginx-build/nginx-clojure-master/src/c/ngx_http_clojure_shared_map_hashmap.c
			
			
				#maybe this is needed:
					apt install leiningen
					cd /root/nginx-build/nginx-clojure-master/
					lein jar
		
		
			to get java started on nginx start :
				nginx config
				http {
				......
					jvm_handler_type 'java'; # or handler_type 'groovy'
					jvm_init_handler_name 'my.test/InitHandler'; 
				....
				}
				java class:
					public static class JVMInitHandler implements NginxJavaRingHandler {
						@Override
						public Object[] invoke(Map<String, Object> ctx) {
							NginxClojureRT.log.info("JVMInitHandler invoked!");
							return null; // or return new Object[] {500, null, null}; for  an error
						}
					}
				
				
				hello world response handler:
					package mytest;
					import static nginx.clojure.MiniConstants.*;

					import java.util.HashMap;
					import java.util.Map;
					public  class Hello implements NginxJavaRingHandler {

							@Override
							public Object[] invoke(Map<String, Object> request) {
								return new Object[] { 
										NGX_HTTP_OK, //http status 200
										ArrayMap.create(CONTENT_TYPE, "text/plain"), //headers map
										"Hello, Java & Nginx!"  //response body can be string, File or Array/Collection of them
										};
							}
						}
				location /myJava {
				  content_handler_type 'java';
				  content_handler_name 'mytest.Hello';
				}
		
		#compile and install zlib library
			wget http://zlib.net/zlib-1.2.11.tar.gz
			tar -zxf zlib-1.2.11.tar.gz
			cd zlib-1.2.11
			./configure
			make
			sudo make install
			
			
		cd /root/nginx-build/nginx-1.15.2/
		./configure --with-openssl=/root/openssltemp/openssl-1.1.1-pre8 --with-openssl-opt=enable-tls1_3 --with-http_realip_module  --with-http_v2_module --prefix=/var/jetendo-server/nginx --user=nginx --group=nginx --with-http_ssl_module --with-http_gzip_static_module  --with-http_flv_module --with-http_mp4_module --with-http_stub_status_module --add-module=/root/nginx-build/nginx-upload-module-master  --add-module=/root/nginx-build/ngx_devel_kit-master --add-module=/root/nginx-build/set-misc-nginx-module-master --add-module=/root/nginx-build/nginx-clojure-master/src/c
		
		# didn't build: 
		make
		make install
		cd /var/jetendo-server/nginx
		mkdir cache client_body_temp fastcgi_temp proxy_temp scgi_temp uwsgi_temp ssl upload_temp
		chown www-data:root cache client_body_temp fastcgi_temp proxy_temp scgi_temp uwsgi_temp upload_temp
		chmod 770 cache client_body_temp fastcgi_temp proxy_temp scgi_temp uwsgi_temp upload_temp
		chmod -R 400 ssl
		mkdir /var/jetendo-server/nginx/conf/sites/
		mkdir /var/jetendo-server/nginx/conf/sites/jetendo/
		chmod -R 770 /var/jetendo-server/nginx/conf/sites
		
		# add mysql to www-data group so lucee / mysql backups work.
		usermod -G mysql,www-data mysql
		
		# service is not running until symbolic link and reboot steps are followed below

		openssl dhparam -out /var/jetendo-server/nginx/ssl/dh2048.pem -outform PEM -2 2048
		
	add mime-types to /var/jetendo-server/nginx/conf/mime.types
		
		audio/webm weba;
		application/x-font-ttf             ttf;
		font/opentype                      otf;
		
	
	
Install lucee
	Compile and Install Apache APR Library
		mkdir /var/jetendo-server/system/apr-build/
		cd /var/jetendo-server/system/apr-build/
		# get the newest apr unix gz here: http://apr.apache.org/download.cgi
		wget http://apache.mirrors.tds.net//apr/apr-1.6.3.tar.gz
		tar -xvf apr-1.6.3.tar.gz
		cd apr-1.6.3
		./configure
		make && make install
	Compile and Install Tomcat Native Library
		export JAVA_HOME=`type -p javac|xargs readlink -f|xargs dirname|xargs dirname`
		cd /var/jetendo-server/system/apr-build/
		# get the newest tomcat native library source here: http://tomcat.apache.org/download-native.cgi
		wget http://mirrors.gigenet.com/apache/tomcat/tomcat-connectors/native/1.2.17/source/tomcat-native-1.2.17-src.tar.gz
		tar -xvzf tomcat-native-1.2.17-src.tar.gz
		cd tomcat-native-1.2.17-src/native/
		./configure --with-apr=/usr/local/apr/bin/ --with-ssl=/root/openssltemp/openssl-1.1.1-pre8 --with-java-home=/usr/lib/jvm/java-11-openjdk-amd64
		make && make install
		
	Install tomcat 9.0.10 manually instead of lucee installer?
		https://www.digitalocean.com/community/tutorials/install-tomcat-9-ubuntu-1804
		cd /var/jetendo-server/system/lucee/temp
		wget http://mirror.cc.columbia.edu/pub/software/apache/tomcat/tomcat-9/v9.0.10/bin/apache-tomcat-9.0.10.tar.gz
		mkdir /var/jetendo-server/tomcat
		tar xzvf /var/jetendo-server/system/lucee/temp/apache-tomcat-9*.tar.gz -C /var/jetendo-server/tomcat --strip-components=1
		cd /var/jetendo-server/tomcat
		chown -R www-data:www-data /var/jetendo-server/tomcat/
		chmod -R 770 /var/jetendo-server/tomcat/ 
		vi /etc/systemd/system/tomcat.service
		 
[Unit]
Description=Apache Tomcat Web Application Container
After=network.target

[Service]
Type=forking

Environment=JAVA_HOME=/usr/lib/jvm/java-10-oracle
Environment=CATALINA_PID=/var/jetendo-server/tomcat/temp/tomcat.pid
Environment=CATALINA_HOME=/var/jetendo-server/tomcat
Environment=CATALINA_BASE=/var/jetendo-server/tomcat
# might have to disable javaagent if it fails temporarily
Environment='CATALINA_OPTS=-Xms512M -Xmx1268M -server -XX:+UseParallelGC -Djava.library.path=/usr/local/apr/lib -javaagent:/var/jetendo-server/tomcat/lucee-server/context/lucee-external-agent.jar'
Environment='JAVA_OPTS=-Djava.awt.headless=true -Djava.security.egd=file:/dev/./urandom'
# fix java 9/10 error in Lucee Admin
Environment='JDK_JAVA_OPTIONS=--add-opens=jdk.management/com.sun.management.internal=ALL-UNNAMED'

ExecStart=/var/jetendo-server/tomcat/bin/startup.sh
ExecStop=/var/jetendo-server/tomcat/bin/shutdown.sh

User=tomcat
Group=tomcat
UMask=0007
RestartSec=10
Restart=always

[Install]
WantedBy=multi-user.target

	update root certificates cacert
		cd /usr/local/share/ca-certificates/
		wget http://curl.haxx.se/ca/cacert.pem
		update-ca-certificates
		dpkg-reconfigure ca-certificates
			say yes and ok.
			
		add this to /var/jetendo-server/system/php/jetendo.ini
			[curl]
			; A default value for the CURLOPT_CAINFO option. This is required to be an
			; absolute path.
			curl.cainfo ="/usr/local/share/ca-certificates/cacert.pem"
		service php7.2-fpm restart
		 
		for tomcat 9 and java 10, use this command instead and say yes:
			/usr/bin/keytool -import -keystore /usr/lib/jvm/java-11-openjdk-amd64/lib/security/cacerts -trustcacerts -file /usr/local/share/ca-certificates/cacert.pem -storepass changeit
		
		 
		
		// replace server.xml and web.xml with the lucee ones which use the luceevhosts directory instead of lucee 
		// upload latest lucee.org snapshot jar to /var/jetendo-server/tomcat/lib/
		systemctl daemon-reload
		systemctl restart tomcat
		systemctl enable tomcat
		visit server and web admin to setup the rest including the first time password
		http://dev.127.0.0.2.nip.io:8888/lucee/admin/server.cfm
		 
		Edit /var/jetendo-server/tomcat/bin/catalina.sh
			Immediately after "echo "Tomcat started."", add these lines where company domain is one of the main domain setup in jetendo for the server administrator.
				
				IMPORTANT: Make sure to change these to your test domain if you aren't using nip.io
				
				Test Server:
					printf "\nLoading Jetendo Application\n"
					/usr/bin/wget -O- 'http://dev.127.0.0.2.nip.io:8888/zcorerootmapping/index.cfm?_zsa3_path=/&zcoreRunFirstInit=1'
					printf "\n[DONE]"
				Production Server:
					printf "\nLoading Jetendo Application\n"
					/usr/bin/wget -O- 'http://dev.127.0.0.1.nip.io:8888/zcorerootmapping/index.cfm?_zsa3_path=/&zcoreRunFirstInit=1'
					printf "\n[DONE]"
			
		# prevent lucee from starting on boot (requires that jetendo-start.php init script is installed and configured - documentation is incomplete for this)
		systemctl disable lucee_ctl
	#	echo manual | sudo tee /etc/init/lucee_ctl.override
		
		This forces the request that loads jetendo to occur on each restart of Lucee.
		
	/var/jetendo-server/lucee/tomcat/bin/setenv.sh
	# adjust Xmx high as you can afford, but at least 512m is necessary
		CATALINA_OPTS="-server -Dsun.io.useCanonCaches=false -Xms512m -Xmx1024m -javaagent:lib/lucee-inst.jar  -Djava.library.path=/usr/local/apr/lib -XX:+OptimizeStringConcat -XX:+UseTLAB -XX:+UseBiasedLocking -Xverify:none -XX:+UseThreadPriorities  -XX:+UseFastAccessorMethods -XX:-UseLargePages -XX:+UseCompressedOops";
		
	Put newest JRE In lucee:
		service lucee_ctl stop
		mv /var/jetendo-server/lucee/jdk/jre /var/jetendo-server/lucee/jdk/jre-original
		mkdir /var/jetendo-server/lucee/jdk/jre
		/bin/cp -rf /usr/lib/jvm/java-10-oracle/jre/* /var/jetendo-server/lucee/jdk/jre
		chown -R www-data:www-data /var/jetendo-server/lucee/
		chmod -R 770 /var/jetendo-server/lucee/

	mkdir /var/jetendo-server/jetendo/
	mkdir /var/jetendo-server/jetendo/sites
	mkdir /var/jetendo-server/jetendo/share
	touch /var/jetendo-server/jetendo/share/hostmap.conf
	touch /var/jetendo-server/jetendo/share/hostmap-redirect.conf
	mkdir /var/jetendo-server/luceevhosts/
	mkdir /var/jetendo-server/luceevhosts/server/
	mkdir /var/jetendo-server/luceevhosts/tomcat-logs/
	cp -rf /var/jetendo-server/lucee/lib/* /var/jetendo-server/luceevhosts/server/
	chown -R www-data:www-data /var/jetendo-server/luceevhosts/
	chmod -R 770 /var/jetendo-server/luceevhosts/
 
	# install the server.xml for production or development
		# development
		cp /var/jetendo-server/system/lucee/server-development.xml /var/jetendo-server/tomcat/conf/server.xml
		cp /var/jetendo-server/system/lucee/web-development.xml /var/jetendo-server/tomcat/conf/web.xml
		cp /var/jetendo-server/system/lucee/logging-development.properties /var/jetendo-server/tomcat/conf/logging.properties
		cp /var/jetendo-server/system/lucee/setenv-development.sh /var/jetendo-server/tomcat/bin/setenv.sh
		# production
		cp /var/jetendo-server/system/lucee/server-production.xml /var/jetendo-server/tomcat/conf/server.xml
		cp /var/jetendo-server/system/lucee/web-production.xml /var/jetendo-server/tomcat/conf/web.xml
		cp /var/jetendo-server/system/lucee/logging-production.properties /var/jetendo-server/tomcat/conf/logging.properties
		cp /var/jetendo-server/system/lucee/setenv-production.sh /var/jetendo-server/tomcat/bin/setenv.sh
		
	
	vi /etc/logrotate.d/tomcat
	/var/jetendo-server/tomcat/logs/catalina.out {
		copytruncate
		daily
		rotate 7
		compress
		missingok
		size 5M
	}
	
	vi /etc/logrotate.d/jetendo
	/var/jetendo-server/jetendo/share/task-log/cfml-tasks.log {
		su root www-data
		copytruncate
		daily
		rotate 7
		compress
		missingok
		size 5M
	}
	
	
	service lucee_ctl start
	
	http://dev.com.127.0.0.2.nip.io:8888/lucee/admin/web.cfm?action=resources.mappings
	 
	 
Install node.js 0.12.x using nodesource PPA
	apt-get install apt-transport-https
	wget -qO- https://deb.nodesource.com/setup_0.12 | bash -
	apt-get install nodejs
	apt-get install build-essential
	
	# install handlebars globally to allow template precompilation
	npm install handlebars -g
	
	node -v
	handlebars -v
	 
Download Install Newest Intel Ethernet Adapter Driver If Production Server Use Intel Device
	lspci | grep -i eth
	
Install Optional Packages If You Want Them:
	# provide KVM virtual machines on production server
		apt-get install cpu-checker qemu-kvm libvirt-bin virtinst bridge-utils ubuntu-virt-server python-vm-builder
	# provide regular ftp
		apt-get install vsftpd
	# provides ab (apachebench) benchmarking utility
		apt-get install apache2-utils
	# provides hard drive smart status utilities
		apt-get install smartmontools
	# utilities for monitoring network and hard drive performance
		apt-get install sysstat iftop
	


# dev server manually modified files
	
Configure the variables in jetendo.ini manually
	/var/jetendo-server/system/php/jetendo.ini
	
Make sure the jetendo.ini symbolic link is created:
	ln -sfn /var/jetendo-server/system/php/jetendo.ini /etc/php/7.2/mods-available/jetendo.ini
Enable the php configuration module:	
	phpenmod jetendo
	service php7.2-fpm restart
	
# development server symbolic link configuration
	ln -sfn /var/jetendo-server/system/nginx-conf/nginx-development.conf /var/jetendo-server/nginx/conf/nginx.conf
	ln -sfn /var/jetendo-server/system/jetendo-sysctl-development.conf /etc/sysctl.d/jetendo-sysctl-development.conf
	ln -sfn /var/jetendo-server/system/monit/jetendo-development.conf /etc/monit/conf.d/jetendo.conf
	ln -sfn /var/jetendo-server/system/apache-conf/development-sites-enabled /etc/apache2/sites-enabled
	ln -sfn /var/jetendo-server/system/php/development-pool /etc/php/7.2/fpm/pool.d
		
	replace sa.your-company.com 3 times in /var/jetendo-server/system/monit/jetendo-development.conf with the jetendo server manager domain and change to https if you are using https 
	
# production server symbolic link configuration
	ln -sfn /var/jetendo-server/system/jetendo-mysql-production.cnf /etc/mysql/conf.d/jetendo-mysql-production.cnf
	ln -sfn /var/jetendo-server/system/nginx-conf/nginx-production.conf /var/jetendo-server/nginx/conf/nginx.conf
	ln -sfn /var/jetendo-server/system/jetendo-sysctl-production.conf /etc/sysctl.d/jetendo-sysctl-production.conf
	ln -sfn /var/jetendo-server/system/monit/jetendo-production.conf /etc/monit/conf.d/jetendo.conf
	ln -sfn /var/jetendo-server/system/apache-conf/production-sites-enabled /etc/apache2/sites-enabled
	ln -sfn /var/jetendo-server/system/php/production-pool /etc/php/7.2/fpm/pool.d
	
	replace sa.your-company.com 3 times in /var/jetendo-server/system/monit/jetendo-production.conf with the jetendo server manager domain and change to https if you are using https 
	
	
cp /var/jetendo-server/system/jetendo-nginx-init /etc/init.d/nginx
chmod +x /etc/init.d/nginx
/usr/sbin/update-rc.d -f nginx defaults

# enable apparmor profiles:
	development server:
		cp -f /var/jetendo-server/system/apparmor.d/development/* /etc/apparmor.d/
		apparmor_parser -r /etc/apparmor.d/
	production server:
		cp -f /var/jetendo-server/system/apparmor.d/production/* /etc/apparmor.d/
		apparmor_parser -r /etc/apparmor.d/
	
	configure the profiles to be specific to your application by editing them in /etc/apparmor.d/ directly.
	
# generate self-signed ssl certs for development
	cd /var/jetendo-server/nginx/ssl/
	openssl genrsa -out dev.com.key 2048
	openssl rsa -in dev.com.key -out dev.com.pem
	openssl req -new -key dev.com.key -out dev.com.csr
	openssl x509 -req -days 3650 -in dev.com.csr -signkey dev.com.key -out dev.com.crt
	chmod -R 400 /var/jetendo-server/nginx/ssl/

# increase security limits
	vi /etc/security/limits.conf
* soft nofile 32768
* hard nofile 32768
root soft nofile 32768
root hard nofile 32768
* soft memlock unlimited
* hard memlock unlimited
root soft memlock unlimited
root hard memlock unlimited
* soft as unlimited
* hard as unlimited
root soft as unlimited
root hard as unlimited

# reboot system to have all changes take effect.	
reboot

Setup Git options
	git config --global user.name "Your Name Here"
	git config --global user.email "your_email@example.com"
	git config --global core.filemode false

	
Configure fail2ban:
	change max retry to 5 and ban time to 600 seconds in the /etc/fail2ban/jail.conf
	service fail2ban restart
	If you are ever blocked from ssh login, restarting the server or waiting 10 minutes will allow you back in.

Configure Postfix to use Sendgrid.net for relying mail.
	vi /etc/aliases,  Find the line for "root" and make it "root: EMAIL_ADDRESS" where EMAIL_ADDRESS is the email address that system & security related emails should be forwarded to.
	Then run "newaliases"
	
	comment out line starting with "relayhost" in /etc/postfix/main.cf
	
	Relay mail with Sendgrid.net (Optional, but recommended for production servers)
		Add this to the end /etc/postfix/main.cf where your_username and your_password are replaced with the sendgrid.net login information.
			# jetendo-custom-smtp-begin
			smtp_sasl_auth_enable = yes
			smtp_sasl_password_maps = static:your_username:your_password
			smtp_sasl_security_options = noanonymous
			smtp_tls_security_level = may
			header_size_limit = 4096000
			relayhost = [smtp.sendgrid.net]:587
			# jetendo-custom-smtp-end
		
	Or relay mail to a google account by adding the following to 
		vi /etc/postfix/main.cf
			#jetendo-custom-smtp-begin
			relayhost = [smtp.gmail.com]:587
			smtp_sasl_auth_enable = yes
			smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
			smtp_sasl_security_options = noanonymous
			smtp_tls_CAfile = /etc/postfix/cacert.pem
			smtp_use_tls = yes
			# jetendo-custom-smtp-end
		vi /etc/postfix/sasl_passwd
			[smtp.gmail.com]:587    USERNAME@gmail.com:PASSWORD
		chmod 400 /etc/postfix/sasl_passwd
		postmap /etc/postfix/sasl_passwd
		cat /etc/ssl/certs/Thawte_Premium_Server_CA.pem | sudo tee -a /etc/postfix/cacert.pem
	
	After changing the postfix configuration, restart the service:
		service postfix reload
		
	Verify the mail service is working, by logging in to the guest machine with SSH and typing the following command:
		echo "Test email" | mailx -s "Hello world" your_email@your_company.com
		
	If the mail service isn't working, make sure you entered the right information and followed the steps correctly.
	
	If the problem persists, check the logs at /var/log/mail.log or /var/log/syslog for error messages.
	
Enable hardware random number generator on non-virtual machine.  This is not safe on a virtual machine.
	rngd -r /dev/urandom
	
	on virtual machine use this instead:
		apt-get install haveged
	
manually download the latest 64-bit stable linux version of wkhtmltopdf on the website: http://wkhtmltopdf.org/downloads.html
	wget https://downloads.wkhtmltopdf.org/0.12/0.12.5/wkhtmltox_0.12.5-1.bionic_amd64.deb
	apt-get install xfonts-base xfonts-75dpi
	dpkg -i /root/wkhtmltox_0.12.5-1.bionic_amd64.deb
	
	
Configure Jungledisk (Optional)
	This is a recommend solution for remote backup of production servers.
	
	Install Jungledisk
		Download 64-bit Server Edition Software from your jungledisk.com account:
		cd /root/
		wget https://downloads.jungledisk.com/jungledisk/junglediskserver_316-0_amd64.deb
		
		# Run this command to install it.  Make sure the file name matches the file you downloaded.
		dpkg -i /root/junglediskserver_316-0_amd64.deb
		
		Reset the license key on your jungledisk.com account page and replace LICENSE_KEY below with the key they generated for you.
		vi /etc/jungledisk/junglediskserver-license.xml
			<?xml version="1.0" encoding="utf-8"?><configuration><LicenseConfig><licenseKey>LICENSE_KEY</licenseKey><proxyServer><enabled>0</enabled> <proxyServer></proxyServer><userName></userName><password></password></proxyServer></LicenseConfig></configuration>

			
		need to exclude these from being backed up
			/var/hotcopy   # this is for r1soft cdp
			/var/cache
			/var/jetendo-server/mysql/data/   # make sure /zbackup/mysql is backed up!
			/usr
			/sbin
			/lib
			/lib64
			/bin
			/boot
			/dev
			/proc
			
		Change cache directory to:
			/zbackup/jungledisk/cache/
			
		service junglediskserver restart
	Use the management client interface from https://www.jungledisk.com/downloads/business/server/linux/ to further configure what and when to backup.  It is highly recommended you enable the encrypted backup feature for best security.  Be sure not to lose your decryption password.

Configuring Static IPs
	vi /etc/network/interfaces
	
	# Be careful, you may accidentally take your server off the Internet if you make a mistake.  It is best to do this with KVM access or have the hosting company help you.
	
	# By default, Virtualbox is configured to use NAT, and this configuration looks like this in /etc/netplan/50-cloud-init.yaml after installing ubuntu
		network:
			ethernets:
				enp0s3:
					addresses: []
					dhcp4: true
			version: 2

	
	TODO: NEED NEW netplan SYNTAX FOR LIVE SERVER: # To replace NAT with a static IP for same interface, delete "auto eth0" and "iface eth0 inet dhcp" and use the settings below.  Make sure the IPs match what is provided by your ISP.  The DNS Nameservers should ideally be your ISP's nameservers for best performance or google public dns which is: 8.8.8.8 8.8.4.4
		auto eth0
		iface eth0 inet static
		address 192.168.0.2
		netmask 255.255.255.0
		network 192.168.0.0
		broadcast 192.168.0.255
		gateway 192.168.0.1
		dns-nameservers 192.168.0.1 192.168.0.1
	
	This is the static ip configuration for development server:
		auto eth0
		iface eth0 inet static
			address 10.0.2.15
			netmask 255.255.255.0
			network 10.0.2.0
			broadcast 10.0.2.255
			gateway 10.0.2.2
			dns-nameservers 10.0.2.2
			
		auto eth0:1
		iface eth0:1 inet static
			address 10.0.2.16
			netmask 255.255.255.0
			network 10.0.2.0
			broadcast 10.0.2.255

	# each additional ip appends to the interface name, a colon and a sequential number.  Such as p4p1:1 below.  You can add as many of these as you wish to a single interface.  It is not necessary to specify the dns-nameservers again.
	auto p4p1:1
	iface p4p1:1 inet static
		address 192.168.0.3
		netmask 255.255.255.0
		network 192.168.0.0
		broadcast 192.168.0.255

Visit http://nip.io/ to understand how this free service helps you create development environments with minimal re-configuration.
	Essentially it automates dns configuration, to let you create new domains instantly that point to any ip address you desire.
	http://mydomain.com.127.0.0.1.nip.io/ would attempt to connection to 127.0.0.1 with the host name mydomain.com.127.0.0.1.nip.io. 
	Jetendo has been designed to support this service by default.
	
	You can also use nip.io the same way.
	
By default, this is not needed.  If you want additional pools, add them like this.  one listen path for each fastcgi pool.   /etc/php/7.2/fpm/pool.d/dev.com.conf  - but lets symbolic link it to /var/jetendo-server/system/php/fpm-pool-conf/
			[dev.com]
			listen = /var/jetendo-server/php/run/fpm.dev.com.sock
			listen.owner = www-user
			listen.mode = 0600
			user = devsite1
			group = www-data
			pm = dynamic
			pm.max_children = 5
			pm.min_spare_servers = 1
			pm.max_spare_servers = 2
	useradd -g www-data devsite1
	mkdir /var/www/devsite1
	chown devsite1:www-data /var/www/devsite1
	
	for devsite1.com, nginx uses
		fastcgi_pass unix:/var/jetendo-server/php/run/fpm.devsite1.sock;

speed up shutdown by disabling network printing service
	systemctl disable cups-browsed.service
		
Reboot the virtual machine to ensure all services are installed and running before continuing with Jetendo CMS installation
	At a shell prompt, type: 
		reboot
	Also, to turn off the machine gracefully, you can type the following at a shell prompt:
		poweroff
	
		 
Configure Jetendo CMS

	Install the Jetendo source code from git by running the php script below from the command line.
	You can edit this file to change the git repo or branch if you want to work on a fork or different branch of the project.  If you intend to contribute to the project, it would be wise to create a fork first.  You can always change your git remote origin later.
	Note: If you want to run a RELEASE version of Jetendo CMS, skip running this file.
		php /var/jetendo-server/system/install-jetendo.php
		
		or just run these commands:
		chmod 755 /var/jetendo-server/jetendo/
		chmod 770 /var/jetendo-server/jetendo/sites
		mkdir /var/jetendo-server/jetendo/themes
		mkdir /var/jetendo-server/jetendo/sites-writable
		chmod 770 /var/jetendo-server/jetendo/sites-writable
		mkdir /var/jetendo-server/backup/
		
	Add the following mappings to the Lucee web admin for the /var/jetendo-server/jetendo/ context:
		Lucee web admin URL for VirtualBox (create a new password if it asks.)
		
		http://dev.com.127.0.0.2.nip.io:8888/lucee/admin/web.cfm?action=resources.mappings
	
		The resource path for "/zcorecachemapping" must be the sites-writable path for the adminDomain.
		For example, if request.zos.adminDomain = "http://jetendo.your-company.com";
		Then the correct configuration is:
			Virtual: /zcorecachemapping
			Resource Path: /var/jetendo-server/jetendo/sites-writable/jetendo_your-company_com/_cache
		
		Virtual: /zcorerootmapping
		Resource Path: /var/jetendo-server/jetendo/core
		After creating "/zcorerootmapping", click the edit icon and make sure "Web accessible" is checked and click save.
		
		Virtual: /jetendo-themes
		Resource Path: /var/jetendo-server/jetendo/themes
		
		Virtual: /jetendo-sites-writable
		Resource Path: /var/jetendo-server/jetendo/sites-writable
		
		Virtual: /jetendo-database-upgrade
		Resource Path: /var/jetendo-server/jetendo/database-upgrade
	
	Setup the Jetendo datasource - the database, datasource, jetendo_datasource, and request.zos.zcoreDatasource must all be the same name.
		http://dev.com.127.0.0.2.nip.io:8888/lucee/admin/web.cfm?action=services.datasource
		Add mysql datasource named "jetendo" or whatever you've configured it to be in the jetendo config files.
			host: 127.0.0.1
			Required options: 
				Blog: Check
				Clob: Check
				Use Unicode: true
				Alias handling: true
				Allow multiple queries: false
				Zero DateTime behavior: convertToNull
				Auto reconnect: false
				Throw error upon data truncation: false
				TinyInt(1) is bit: false
				Legacy Datetime Code: true

	
	Enable complete null support and set Key case to Preserve Case (fixes javascript case problems) and Template charset to UTF-8:
		http://dev.com.127.0.0.2.nip.io:8888/lucee/admin/server.cfm?action=server.compiler
		
	Enable mail server:
		http://dev.com.127.0.0.2.nip.io:8888/lucee/admin/server.cfm?action=services.mail
		
		Under Mail Servers -> Server (SMTP), type "localhost" and click update"
		
	Configure Lucee security sandbox - Warning: Upgrading Lucee has caused the security sandbox to be deleted in the past.  Be sure to review it periodically.
		http://jetendo.your-company.com.127.0.0.2.nip.io:8888/lucee/admin/server.cfm?action=security.access&sec_tab=SPECIAL
		Under Create new context, select "b180779e6dc8f3bb6a8ea14a604d83d4 (/var/jetendo-server/jetendo/sites)" and click Create
		Then click edit next to the specific web context
		On a production server, set General Access for read and write to "closed" when you don't need to access the Lucee admin.   You can re-enable it only when you need to make changes.
		Under File Access, select "Local" and enter the following directories. 
			Note: In Lucee 4.2, you have to enter one directory at a time by submitting the form with one entered, and then click edit again to enter the next one.
			/var/jetendo-server/jetendo/core
			/var/jetendo-server/jetendo/sites
			/var/jetendo-server/jetendo/share
			/var/jetendo-server/jetendo/execute
			/var/jetendo-server/jetendo/public
			/var/jetendo-server/luceevhosts/1599b2419bcff43008448d60f69f646e/
			/var/jetendo-server/jetendo/sites-writable
			/var/jetendo-server/jetendo/themes
			/var/jetendo-server/jetendo/database-upgrade
			/var/jetendo-server/backup/
			/var/jetendo-server/lucee/tomcat/lucee-server/context/
			/var/jetendo-server/lucee/tomcat/lucee-server/context/userdata
			/zbackup/backup
			/zbackup/jetendo
			
			# if you have other drives, you can do something like this too:
			/zbackup2/backup
			/zbackup2/jetendo
		Uncheck "Direct Java Access"
		Uncheck all the boxes under "Tags & Functions" - Jetendo CMS intentionally allows not using these features to be more secure.
		
	Edit the values in the following files to match the configuration of your system.
		touch /var/jetendo-server/jetendo/core/config.cfc
		chown www-data:www-data /var/jetendo-server/jetendo/core/config.cfc
		chmod 770 /var/jetendo-server/jetendo/core/config.cfc
	If you want to run a RELEASE version of Jetendo CMS, follow these steps:
		Download the release file for the "jetendo" project, and unzip its contents to /var/jetendo-server/jetendo in the virtual machine or server.  Make sure that there is no an extra /var/jetendo-server/jetendo/jetendo directory.  The files should be in /var/jetendo-server/jetendo/
		Download the release file for the "jetendo-default-theme" project and unzip its contents to /var/jetendo-server/jetendo/themes/jetendo-default-theme in the virtual machine or server. Make sure that there is no an extra /var/jetendo-server/jetendo/themes/jetendo-default-theme/jetendo-default-theme directory.  The files should be in /var/jetendo-server/jetendo/themes/jetendo-default-theme
		chmod -R 770 /var/jetendo-server/jetendo/themes
		chown -R www-data:www-data /var/jetendo-server/jetendo/themes
		
		vi /var/jetendo-server/custom-secure-scripts
		vi /var/jetendo-server/custom-secure-scripts/retsConfig.php
			<?php  
			$arrRetsConfig=array();
			?>
		
		Run this command to install the release without forcing it to use the git repository:
			php /var/jetendo-server/jetendo/scripts/install.php disableGitIntegration
			
			If it says "configPath is missing", then you haven't setup php with jetendo.ini correctly yet.
			
		Note: the project will not be installed as a git repository, so you will have to manually perform upgrades in the future.
		
	If you want to run the DEVELOPMENT version of Jetendo CMS, follow these steps:
		Run this command to install the Jetendo CMS cron jobs and verify the integrity of the source code.
			php /var/jetendo-server/jetendo/scripts/install.php
		Any updates since the last time you ran this installation file, will be pulled from github.com.
		Note: The project will be installed as a git respository.
		
	At the end of a successful run of install.php, you'll be told to visit a URL in the browser to complete installation.  The first time you run that URL, it will restore the database tables, and verify the integrity of the installation.  Please be patient as this process can take anywhere from 10 seconds to a couple minutes the first time depending on your environment.
	
	Troubleshooting Tip: If you have a problem during this step, you may need to drop the entire database, and restart the Lucee Server after correcting the configuration.   This is because the first time database installation may fail if you make a mistake in your configuration or if there is a bug in the install script.  Please make us aware of any problems you encountered during installation so we can improve the software.
	
	After it finishes loading, a page should appear saying "You're Almost Ready".
	
	You will be prompted to create a server administrator user and set some other information.
	
	Make sure you select the 127.0.0.1 as the ip on a development machine unless you know what you're doing.
	
	Make sure to remember the email and password you use for the server administrator account.  If you do lose your login, you can reset the password via email.
	
	Once that is done, you will be prompted to login to the server manager and begin using Jetendo CMS.
	
Install KernelCare.com (paid service for no-reboot kernel updates)
		wget https://downloads.kernelcare.com/kernelcare-latest.deb
		dpkg -i kernelcare-latest.deb
		# set license key (where KEY is the actual key you purchased)
		/usr/bin/kcarectl --register KEY
		# To check if patches applied: 
		/usr/bin/kcarectl --info
		# The software will automatically check for new patches every 4 hours. If you would like to run update manually: 
		/usr/bin/kcarectl --update

Preparing the virtual machine for distribution:
	Run these commands inside the virtual machine - it will automatically poweroff when complete.
		killall -9 php
		php /var/jetendo-server/system/clean-machine.php
	In host, run compact on the vdi file - Windows command line example:
		"C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" modifyhd jetendo-server-os.vdi --compact
	# The VDI files should be less then 4gb afterwards.
	# Manually 7-zip the virtual machine - It takes about 10 minutes to make it 6 times smaller
	# regular zip takes 5 minutes to make it 5 times smaller
		jetendo-server-os.vdi, jetendo-server-swap.vdi and jetendo-server.vbox
	