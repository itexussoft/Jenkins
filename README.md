# Continuous integration with Jenkins


In this manual we're going to configure [Jenkins](https://jenkins.io/) for testing rails application  (backed w/ PostgreSQL).

## Install Jenkins

Install Jenkins on your server:
```bash
wget -q -O - https://pkg.jenkins.io/debian/jenkins-ci.org.key | sudo apt-key add -
sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt-get update
sudo apt-get install jenkins
```

#### Note

If you have Error `E: Sub-process /usr/bin/dpkg returned an error code (1)`, you need to install `oracle-java7-installer`.
```bash
sudo add-apt-repository ppa:webupd8team/java
sudo apt-get update
sudo apt-get install oracle-java7-installer
```

I have a localization error while installing:
```bash
perl: warning: Setting locale failed.
perl: warning: Please check that your locale settings:
	LANGUAGE = (unset),
	LC_ALL = (unset),
	LC_TIME = "ru_RU.UTF-8",
	LC_MONETARY = "ru_RU.UTF-8",
	LC_ADDRESS = "ru_RU.UTF-8",
	LC_TELEPHONE = "ru_RU.UTF-8",
	LC_NAME = "ru_RU.UTF-8",
	LC_MEASUREMENT = "ru_RU.UTF-8",
	LC_IDENTIFICATION = "ru_RU.UTF-8",
	LC_NUMERIC = "ru_RU.UTF-8",
	LC_PAPER = "ru_RU.UTF-8",
	LANG = "en_US.UTF-8"
    are supported and installed on your system.
```
also this error may appear during the `PostgresSQL` installation.

To avoid it change your `/etc/default/locale` file:

Insert 
```bash
LANGUAGE=en_US.UTF-8
LC_ALL=en_US.UTF-8
LANG=en_US.UTF-8
LC_TYPE=en_US.UTF-8
```
and then install `language-pack-en`

```bash
sudo apt-get install language-pack-en
```

## Install Apache

```bash
sudo apt-get install apache2

sudo a2enmod proxy
sudo a2enmod proxy_http
```

Create `/etc/apache2/sites-available/jenkins.conf` file and insert in the him.
```xml
<VirtualHost *:80>
    ServerName HOSTNAME
    ProxyRequests Off
    <Proxy *>
        Order deny,allow
        Allow from all
    </Proxy>
    ProxyPreserveHost on
    ProxyPass / http://localhost:8080/
</VirtualHost>
```
Delete default config in `/etc/apache2/sites-enabled/`.

```bash
sudo rm 000-default.conf 
```
Add Jenkins config to `sites-enabled` and restart Apache.
```bash
sudo a2ensite jenkins
sudo service apache2 restart
```

## Install PostgresSQL, RVM and Git for your user (with root access)

### PostgresSql

```bash
sudo apt-get install postgresql postgresql-contrib postgresql-client postgresql-client-common
```

#### Add database user to PostgresSQL

Enter to `psql` console

```bash
sudo -u postgres psql
```
and create role:
```bash
create role username with createdb login password 'password';
```
You also need change `/etc/postgresql/9.3/main/pg_hba.conf` to use password authentication in database.

```
# TYPE  DATABASE        USER            ADDRESS                 METHOD
from
local   all             all                                     peer
to
local   all             all                                     md5
```

Then restart PostgresSQL server.
```bash
sudo service postgresql restart
```
### Git

```bash
sudo apt-get update
sudo apt-get install git
```

### RVM

```bash
gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3

\curl -sSL https://get.rvm.io | bash -s stable
```
#### Install RVM for Jenkins user

Also we need to install RVM for Jenkins user.
Installing RVM for Jenkins is not different, but we should add `[[ -s "$HOME/.rvm/scripts/rvm" ]] && source "$HOME/.rvm/scripts/rvm"` into `.bashrc` to autorun RVM with Jenkins.

We are installed all necessary for Jenkins to work. Now we can input your instance public IP in browser and see that Jenkins is working.

## Jenkins Config

Enter AdminPassword to begin working with Jenkins.

![](https://i.imgur.com/nkUHJ9O.png)

Select one to start using Jenkins.

![](https://i.imgur.com/haFhgfL.png)

### Install Jenkins Plugins
Jenkins has many plugins to make working easier. You can control them in the tab  `Configuration Jenkins` -> `Management plug-ins` -> `Available`.

We will need these plugins:
+ GitHub plugin
+ Git plugin
+ GitHub Pull Request Builder
+ RubyMetrics plugin for Jenkins

#### Note
When I installed the plugins, I had an error `Caused by: java.lang.OutOfMemoryError: PermGen space`
To solve it is necessary to open file `/etc/default/jenkins` and add arguments: 
```bash
JAVA_ARGS="-Xmx2048m -XX:MaxPermSize=512m -Djava.awt.headless=true"
```

## Create Continuous integration for Rails Application

To create your integration with project click on `New Item`.

Enter a name and select `Freestyle project`.


![](https://i.imgur.com/eCYEmZz.png)


Check `GitHub project` and enter `Project url`. It's necessary to create link to project repository.

![](https://i.imgur.com/KqmMCZ8.png)

In `Source Code Management` check `Git`. Enter your `Repository URL` and `Branch Specifier`.
![](https://i.imgur.com/jsIZ1CA.png)

In `Build Triggers` check `Build when a change is pushed to GitHub`.

In `Build` check `Execute shell` where write command to build application.

```bash
#!/bin/bash -xe
export RAILS_ENV=test
source ~/.bashrc
rvm use 2.2.4@dev --create
gem instal bundler
bundle install
bundle exec rake db:create db:migrate
bundle exec rspec
```
Click on Save.

Click `Build Now` for building project. Now your project was building.

## Create webhook for project repository

[Webhooks](https://developer.github.com/webhooks/) allow you to build or set up integrations which subscribe to certain events.

In Git repository `Setings` select `Webhooks` and `Add webhook`.

![](https://i.imgur.com/9pVbYhJ.png)

In `Payload URL` enter Url and select event (Push, Repository).
![](https://i.imgur.com/MqDaWJ1.png)
![](https://i.imgur.com/aXufrhh.png)

## Add Rcov and Brakeman

You need to install jenkins plugins `RubyMetrics plugin for Jenkins` and `Brakeman Plugin` to be able to use them.

In `Configure` your item on Jenkins add `Post-build Actions`.
Select `Publish Brakeman warnings` and `Publish Rcov report`.

Fill in `Brakeman Output File` for `Publish Brakeman warnings`.

Fill in `coverage/rcov` into `Rcov report directory`.

![](https://i.imgur.com/FZ9bf2Z.png)

In `Execute shell` add 

```bash
gem install brakeman --no-ri --no-rdoc
brakeman -o brakeman-output.tabs --no-progress --separate-models
```
#### Change your Gemfile

```ruby
gem 'brakeman', require: false, group: :development

group :development, :test do
  gem "simplecov", require: false
  gem "simplecov-rcov"
end
```
And finally the following lines at the top of `spec_helper.rb`:
```ruby
require 'simplecov'
require 'simplecov-rcov'
SimpleCov.formatter = SimpleCov::Formatter::RcovFormatter
SimpleCov.start 'rails'
```

## Conclusion

In this post we got familiar with Jenkins installation from scratch and learnt how to integrate a project continuously.

If you have any questions - please share them as issues. Thank you for reading!
