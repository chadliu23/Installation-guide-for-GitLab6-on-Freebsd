Installation-guide-for-GitLab6-on-Freebsd
=========================================

This installation guide was create for and tested on **Freebsd 8.3**. 
his is **NOT** the official installation guide to set up a production server since upstream doesnt support **FreeBSD**. To set up a **development installation** or for many other installation options please consult [the installation section in the readme](https://github.com/gitlabhq/gitlabhq#installation).
# Overview

The GitLab installation consists of setting up the following components:

1. Packages / Dependencies
2. System Users
3. GitLab shell
4. Database
5. GitLab
6. Apache Setup
7. Check Installation

# 1. Packages / Dependencies

install using **ports** 

1. devel/git 

		sudo cd /usr/ports/devel/git && make install clean
       
2. databases/redis

		sudo cd /usr/ports/databases/redis && make install clean
		cp /usr/local/etc/redis.conf.sample /usr/local/etc/redis.conf
		echo "redis_enable="YES"" >> /etc/rc.conf
		service redis start

3. devel/icu

		sudo cd /usr/ports/devel/icu && make install clean

4. textproc/libxml2

		sudo cd /usr/ports/textproc/libxml2 && make install clean

5. mail/postfix 
       
		sudo cd /usr/ports/mail/postfix && make install clean

6. textproc/libxslt

		sudo cd /usr/ports/textproc/libxslt && make install clean

7.  python 

		# Install Python
		sudo cd /usr/ports/lang/python27 && make install clean
		
		# Make sure that Python is 2.5+ (3.x is not supported at the moment)
		python --version

8.  logrotate

		sudo cd /usr/ports/sysutils/logrotate && make install clean
		

9.  docutils

		curl -O http://heanet.dl.sourceforge.net/project/docutils/docutils/0.11/docutils-0.11.tar.gz
		gunzip -c docutils-0.11.tar.gz | tar xopf -
		cd docutils-0.11
		sudo python setup.py install
		
10. mysql 

		sudo cd /usr/ports/databases/mysql55-server && make install clean
		echo "mysql_enable=YES" >> /etc/rc.conf

11. Ruby

		echo DEFAULT_VERSIONS=ruby=1.9 >> /etc/make.conf
		sudo cd /usr/ports/lang/ruby19 && make install clean
		sudo cd /usr/ports/devel/ruby-gems && make install clean
		sudo cd /usr/ports/sysutils/rubygem-bundler && make install clean	
		

# 2. System Users

Create a `git` user and group for Gitlab:

		sudo pw addgroup git
		sudo pw adduser git -g git -m -d /home/git -c "GitLab"

# 3. GitLab shell

GitLab Shell is a ssh access and repository management software developed specially for GitLab.

		cd /home/git
		sudo -u git git clone https://github.com/gitlabhq/gitlab-shell.git
		cd gitlab-shell
		sudo -u git git checkout v1.7.9
		sudo -u git cp config.yml.example config.yml
		cd gitlab-shell
		
Edit config and replace **gitlab_url**

with something like 'http://domain.com/'

edit and make sure that is pointing to **/usr/local/bin/redis-cli**

		sudo -u git -H vim config.yml

Do setup

		sudo -u git -H ./bin/install

# 4. Database

Now login in mysql

		mysql -u root -pPASSWORD    # if first time: mysql -u root -p

Set a password on the anonymous accounts use. (change $password to a real password)

		mysql> SET PASSWORD FOR ''@'localhost' = PASSWORD('$password');

Set a password for the root account. (change $password to a real password)

		mysql> SET PASSWORD FOR 'root'@'localhost' = PASSWORD('$password');

Create a new user for our gitlab setup 'gitlab'

		CREATE USER 'gitlab'@'localhost' IDENTIFIED BY 'PASSWORD_HERE';

Create database

		CREATE DATABASE IF NOT EXISTS `gitlabhq_production` DEFAULT CHARACTER SET `utf8` COLLATE `utf8_unicode_ci`;

Grant the GitLab user necessary permissions on the table.

		GRANT SELECT, LOCK TABLES, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER ON `gitlabhq_production`.* TO 'gitlab'@'localhost';

Quit the database session

		\q

Try connecting to the new database with the new user

		sudo -u git -H mysql -u gitlab -pPASSWORD_HERE -D gitlabhq_production

if you see **'mysql>'**  you are success database setting, quit with `\q`

# 5. GitLab

#### Download Gitlab

	cd /Users/git
	sudo -u git git clone https://github.com/gitlabhq/gitlabhq.git gitlab
	cd gitlab
	sudo -u git git checkout 6-3-stable

#### Configuring GitLab

	sudo -u git cp config/gitlab.yml.example config/gitlab.yml
	
##### edit gitlab.yml carefully and make sure all the setting to match your setup.

change **/usr/bin/git** => **/usr/local/bin/git**

change **localhost**    => **domain.com**
	

Make sure GitLab can write to the `log/` and `tmp/` directories

	sudo chown -R git log/
	sudo chown -R git tmp/
	sudo chmod -R u+rwX  log/
	sudo chmod -R u+rwX  tmp/

Create directories for repositories make sure GitLab can write to them

	sudo -u git mkdir /home/git/repositories
	sudo chmod -R u+rwX  /home/git/repositories/

Create directory for satellites

	sudo -u git mkdir /home/git/gitlab-satellites

Create directories for sockets/pids and make sure GitLab can write to them

	sudo -u git mkdir tmp/pids/
	sudo -u git mkdir tmp/sockets/
	sudo chmod -R u+rwX  tmp/pids/
	sudo chmod -R u+rwX  tmp/sockets/

Create public/uploads directory otherwise backup will fail

	sudo -u git mkdir public/uploads
	sudo chmod -R u+rwX  public/uploads

Fix

	sudo chown -R git:git /Users/git/repositories/
	sudo chmod -R ug+rwX,o-rwx /Users/git/repositories/
	sudo chmod -R ug-s /Users/git/repositories/
	sudo find /home/git/repositories/ -type d -print0 | sudo xargs -0 chmod g+s

Copy the example Unicorn config

	sudo -u git cp config/unicorn.rb.example config/unicorn.rb


Uncomment  `listen "/Users/git/gitlab/tmp/sockets/gitlab.socket", :backlog => 64` in `unicorn.rb`.

Change Listen port (127.0.0.1:8080) to **non-use** port in `unicorn.rb`.

Configure Git global settings for git user, useful when editing via web

	sudo -u git -H git config --global user.name "GitLab"
	sudo -u git -H git config --global user.email "gitlab@domain.com"

Copy rack attack middleware config

	sudo -u git -H cp config/initializers/rack_attack.rb.example config/initializers/rack_attack.rb

Uncomment `config.middleware.use Rack::Attack` in `/Users/git/gitlab/config/application.rb`

Set up logrotate
	
	sudo mkdir /etc/logrotate.d/
	sudo cp lib/support/logrotate/gitlab /etc/logrotate.d/gitlab

#### Gitlab Mysql Config

	sudo -u git cp config/database.yml.mysql config/database.yml
	
##### Edit "secure password" to real password in config/database.yml

#### Install Gems

You need to edit `Gemfile.lock` (`sudo -u git vim Gemfile.lock`) and change the versions 

change **libv8** to **3.16.14.3** (in two places), 

change **therubyracer** to **0.12.0** 

change **underscore-rails** to **1.5.2** (in two places). 

You also need to edit `Gemfile` (`sudo -u git vim Gemfile`) 

change **underscore-rails** to **1.5.2**.

##### Install Bundlers

	sudo gem install bundler
	sudo gem install charlock_holmes --version '0.6.9.4'
	sudo bundle install --deployment --without development test postgres aws

#### Initialize Database and Activate Advanced Features

	sudo -u git -H bash -l -c 'bundle exec rake gitlab:setup RAILS_ENV=production'

Here is your admin login credentials:

	login: admin@local.host
	password: 5iveL!fe

#### Install Init script

	sudo cp /home/git/gitlab/lib/support/init.d/gitlab /usr/local/etc/rc.d/gitlab
	sudo chmod +x /usr/local/etc/rc.d/gitlab
	sudo /usr/local/etc/rc.d/gitlab start
	
Make GitLab start on boot:

	echo "gitlab_enable="YES"" >> /etc/rc.conf

#6.  Apache Setup

Install Apache 

Install Passenger 

	sudo gem install passenger
	sudo passenger-install-apache2-module

Edit apache httpd.conf in usr/local/etc/apache22/httpd.conf

	LoadModule passenger_module /usr/local/lib/ruby/gems/1.9/gems/passenger-4.0.26/buildout/apache2/mod_passenger.so
	PassengerRoot /usr/local/lib/ruby/gems/1.9/gems/passenger-4.0.26
	PassengerDefaultRuby /usr/local/bin/ruby19
	
make sure the files are in the right place

for gitlab apache setting

	<VirtualHost *:80>
		DocumentRoot /home/git/gitlab/public
		RailsBaseURI /
		<Directory /home/git/gitlab/public>
     			Allow from all
     			Options -MultiViews	
		</Directory>
	</VirtualHost>

Finally, restart Apache: 

	sudo /usr/local/etc/rc.d/apache22 restart

#7. Check Installation

Check gitlab-shell

	sudo -u git /Users/git/gitlab-shell/bin/check

Double-check environment configuration

	sudo -u git -H bash -l -c 'bundle exec rake gitlab:env:info RAILS_ENV=production'

Do a thorough check. Make sure everything is green.

	sudo -u git -H bash -l -c 'bundle exec rake gitlab:check RAILS_ENV=production'

The script complained about the init script not being up-to-date don't worry. 


# ldap setting

	ldap:
	enabled: true
	host: 'localhost'
	base: 'dc=chadliu23,dc=org'
	port: 389
	uid: 'uid'
	method: 'plain' # "ssl" or "plain"
	bind_dn: 'for_bindaccunt'
	password: 'PASSWORD'
	allow_username_or_email_login: true
	
## Troubleshooting

1. if you add a new project but error message "repository no found"
	
it is your sidkit didn't start up

	sudo -u git -H bundle exec rake sidekiq:start RAILS_ENV=production
	
2.  if you committed but project page 500, gitlab cannot read your commit log, undefined method `split' for nil:NilClass 

	sudo -u git -H git config --global color.ui false

I have ^[[33m before commit and ^[[m at the end of commit, and it cannot parse this. 

just remove those tag before process commits



## Links and sources 

-https://github.com/CiTroNaK/Installation-guide-for-GitLab-on-OS-X

-https://github.com/gitlab-freebsd/gitlabhq/blob/freebsd/doc/install/installation.md
