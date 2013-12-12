Installation-guide-for-GitLab6-on-Freebsd
=========================================

This installation guide was create for and tested on **Freebsd 8.3. 
his is **NOT** the official installation guide to set up a production server since upstream doesnt support **FreeBSD**. To set up a **development installation** or for many other installation options please consult [the installation section in the readme](https://github.com/gitlabhq/gitlabhq#installation).
# Overview

The GitLab installation consists of setting up the following components:

1. Packages / Dependencies
2. System Users
3. GitLab shell
4. Database
5. GitLab

# 1. Packages / Dependencies

install using ports 

1. devel/git 

		sudo cd /usr/ports/devel/git && make install clean
       
2. databases/redis

		sudo cd /usr/ports/devel/git && make install clean
		cp /usr/local/etc/redis.conf.sample /usr/local/etc/redis.conf
		echo "redis_enable="YES"" >> /etc/rc.conf
		service redis start

3. devel/icu

		sudo cd /usr/ports/devel/git && make install clean

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
		
9.  libxml2

		sudo cd /usr/ports/textproc/libxml2 && make install clean  

10.  docutils

		curl -O http://heanet.dl.sourceforge.net/project/docutils/docutils/0.11/docutils-0.11.tar.gz
		gunzip -c docutils-0.11.tar.gz | tar xopf -
		cd docutils-0.11
		sudo python setup.py install
		
11. mysql 

    sudo cd /usr/ports/databases/mysql55-server && make install clean
    echo "mysql_enable=YES" >> /etc/rc.conf

12. Ruby

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
		
    # Edit config and replace gitlab_url
    # with something like 'http://domain.com/'
    # edit and make sure that is pointing to /usr/local/bin/redis-cli
    su -m git -c "vim config.yml"

    # Do setup
    su -m git -c ./bin/install

# 4. Database

Now login in mysql

		mysql -u root -pPASSWORD    # if first time mysql -u root -p

# Set a password on the anonymous accounts use. (change $password to a real password)

		mysql> SET PASSWORD FOR ''@'localhost' = PASSWORD('$password');

# Set a password for the root account. (change $password to a real password)

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

if you see 'mysql>'  you are success database setting

# 5. GitLab
