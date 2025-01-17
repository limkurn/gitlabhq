# Installation

## Select Version to Install

Make sure you view [this installation guide](https://gitlab.com/gitlab-org/gitlab-ce/blob/master/doc/install/installation.md) from the branch (version) of GitLab you would like to install. In most cases this should be the highest numbered stable branch (example shown below).

![Select latest branch](https://i.imgur.com/Lrdxk1k.png)

If the highest number stable branch is unclear please check the [GitLab Blog](https://www.gitlab.com/blog/) for installation guide links by version.

## Important Notes

This guide is long because it covers many cases and includes all commands you need, this is [one of the few installation scripts that actually works out of the box](https://twitter.com/robinvdvleuten/status/424163226532986880).

This installation guide was created for and tested on **Debian/Ubuntu** operating systems. Please read [doc/install/requirements.md](./requirements.md) for hardware and operating system requirements. If you want to install on RHEL/CentOS we recommend using the [Omnibus packages](https://www.gitlab.com/downloads/).

This is the official installation guide to set up a production server. To set up a **development installation** or for many other installation options please see [the installation section of the readme](https://gitlab.com/gitlab-org/gitlab-ce/blob/master/README.md#installation).

The following steps have been known to work. Please **use caution when you deviate** from this guide. Make sure you don't violate any assumptions GitLab makes about its environment. For example many people run into permission problems because they changed the location of directories or run services as the wrong user.

If you find a bug/error in this guide please **submit a merge request** following the [contributing guide](../../CONTRIBUTING.md).

## Overview

The GitLab installation consists of setting up the following components:

1. Packages / Dependencies
1. Ruby
1. System Users
1. Database
1. Redis
1. GitLab
1. Nginx

## 1. Packages / Dependencies

`sudo` is not installed on Debian by default. Make sure your system is
up-to-date and install it.

    # run as root!
    apt-get update -y
    apt-get upgrade -y
    apt-get install sudo -y

**Note:** During this installation some files will need to be edited manually. If you are familiar with vim set it as default editor with the commands below. If you are not familiar with vim please skip this and keep using the default editor.

    # Install vim and set as default editor
    sudo apt-get install -y vim
    sudo update-alternatives --set editor /usr/bin/vim.basic

Install the required packages (needed to compile Ruby and native extensions to Ruby gems):

    sudo apt-get install -y build-essential zlib1g-dev libyaml-dev libssl-dev libgdbm-dev libreadline-dev libncurses5-dev libffi-dev curl openssh-server redis-server checkinstall libxml2-dev libxslt-dev libcurl4-openssl-dev libicu-dev logrotate python-docutils pkg-config cmake

Make sure you have the right version of Git installed

    # Install Git
    sudo apt-get install -y git-core

    # Make sure Git is version 1.7.10 or higher, for example 1.7.12 or 2.0.0
    git --version

Is the system packaged Git too old? Remove it and compile from source.

    # Remove packaged Git
    sudo apt-get remove git-core

    # Install dependencies
    sudo apt-get install -y libcurl4-openssl-dev libexpat1-dev gettext libz-dev libssl-dev build-essential

    # Download and compile from source
    cd /tmp
    curl -L --progress https://www.kernel.org/pub/software/scm/git/git-2.0.0.tar.gz | tar xz
    cd git-2.0.0/
    make prefix=/usr/local all

    # Install into /usr/local/bin
    sudo make prefix=/usr/local install

    # When editing config/gitlab.yml (Step 5), change the git -> bin_path to /usr/local/bin/git

**Note:** In order to receive mail notifications, make sure to install a mail server. By default, Debian is shipped with exim4 but this [has problems](https://github.com/gitlabhq/gitlabhq/issues/4866#issuecomment-32726573) while Ubuntu does not ship with one. The recommended mail server is postfix and you can install it with:

    sudo apt-get install -y postfix

Then select 'Internet Site' and press enter to confirm the hostname.

## 2. Ruby

The use of ruby version managers such as [RVM](http://rvm.io/), [rbenv](https://github.com/sstephenson/rbenv) or [chruby](https://github.com/postmodern/chruby) with GitLab in production frequently leads to hard to diagnose problems. For example, GitLab Shell is called from OpenSSH and having a version manager can prevent pushing and pulling over SSH. Version managers are not supported and we strongly advise everyone to follow the instructions below to use a system ruby.

Remove the old Ruby 1.8 if present

    sudo apt-get remove ruby1.8

Download Ruby and compile it:

    mkdir /tmp/ruby && cd /tmp/ruby
    curl -L --progress ftp://ftp.ruby-lang.org/pub/ruby/2.1/ruby-2.1.2.tar.gz | tar xz
    cd ruby-2.1.2
    ./configure --disable-install-rdoc
    make
    sudo make install

Install the Bundler Gem:

    sudo gem install bundler --no-ri --no-rdoc

## 3. System Users

Create a `git` user for GitLab:

    sudo adduser --disabled-login --gecos 'GitLab' git

## 4. Database

We recommend using a PostgreSQL database. For MySQL check [MySQL setup guide](database_mysql.md). *Note*: because we need to make use of extensions you need at least pgsql 9.1.

    # Install the database packages
    sudo apt-get install -y postgresql postgresql-client libpq-dev

    # Login to PostgreSQL
    sudo -u postgres psql -d template1

    # Create a user for GitLab.
    template1=# CREATE USER git CREATEDB;

    # Create the GitLab production database & grant all privileges on database
    template1=# CREATE DATABASE gitlabhq_production OWNER git;

    # Quit the database session
    template1=# \q

    # Try connecting to the new database with the new user
    sudo -u git -H psql -d gitlabhq_production

## 5. Redis

    sudo apt-get install redis-server

    # Configure redis to use sockets
    sudo cp /etc/redis/redis.conf /etc/redis/redis.conf.orig

    # Disable Redis listening on TCP by setting 'port' to 0
    sed 's/^port .*/port 0/' /etc/redis/redis.conf.orig | sudo tee /etc/redis/redis.conf

    # Enable Redis socket for default Debian / Ubuntu path
    echo 'unixsocket /var/run/redis/redis.sock' | sudo tee -a /etc/redis/redis.conf

    # Activate the changes to redis.conf
    sudo service redis-server restart

    # Add git to the redis group
    sudo usermod -aG redis git

## 6. GitLab

    # We'll install GitLab into home directory of the user "git"
    cd /home/git

### Clone the Source

    # Clone GitLab repository
    sudo -u git -H git clone https://gitlab.com/gitlab-org/gitlab-ce.git -b 7-3-stable gitlab

**Note:** You can change `7-3-stable` to `master` if you want the *bleeding edge* version, but never install master on a production server!

### Configure It

    # Go to GitLab installation folder
    cd /home/git/gitlab

    # Copy the example GitLab config
    sudo -u git -H cp config/gitlab.yml.example config/gitlab.yml

    # Update GitLab config file, follow the directions at top of file
    sudo -u git -H editor config/gitlab.yml

    # Make sure GitLab can write to the log/ and tmp/ directories
    sudo chown -R git log/
    sudo chown -R git tmp/
    sudo chmod -R u+rwX log/
    sudo chmod -R u+rwX tmp/

    # Create directory for satellites
    sudo -u git -H mkdir /home/git/gitlab-satellites
    sudo chmod u+rwx,g=rx,o-rwx /home/git/gitlab-satellites

    # Make sure GitLab can write to the tmp/pids/ and tmp/sockets/ directories
    sudo chmod -R u+rwX tmp/pids/
    sudo chmod -R u+rwX tmp/sockets/

    # Make sure GitLab can write to the public/uploads/ directory
    sudo chmod -R u+rwX  public/uploads

    # Copy the example Unicorn config
    sudo -u git -H cp config/unicorn.rb.example config/unicorn.rb

    # Find number of cores
    nproc

    # Enable cluster mode if you expect to have a high load instance
    # Ex. change amount of workers to 3 for 2GB RAM server
    # Set the number of workers to at least the number of cores
    sudo -u git -H editor config/unicorn.rb

    # Copy the example Rack attack config
    sudo -u git -H cp config/initializers/rack_attack.rb.example config/initializers/rack_attack.rb

    # Configure Git global settings for git user, useful when editing via web
    # Edit user.email according to what is set in gitlab.yml
    sudo -u git -H git config --global user.name "GitLab"
    sudo -u git -H git config --global user.email "example@example.com"
    sudo -u git -H git config --global core.autocrlf input

    # Configure Redis connection settings
    sudo -u git -H cp config/resque.yml.example config/resque.yml

    # Change the Redis socket path if you are not using the default Debian / Ubuntu configuration
    sudo -u git -H editor config/resque.yml

**Important Note:** Make sure to edit both `gitlab.yml` and `unicorn.rb` to match your setup.

**Note:** If you want to use HTTPS, see [Using HTTPS](#using-https) for the additional steps.

### Configure GitLab DB Settings

    # PostgreSQL only:
    sudo -u git cp config/database.yml.postgresql config/database.yml

    # MySQL only:
    sudo -u git cp config/database.yml.mysql config/database.yml

    # MySQL and remote PostgreSQL only:
    # Update username/password in config/database.yml.
    # You only need to adapt the production settings (first part).
    # If you followed the database guide then please do as follows:
    # Change 'secure password' with the value you have given to $password
    # You can keep the double quotes around the password
    sudo -u git -H editor config/database.yml

    # PostgreSQL and MySQL:
    # Make config/database.yml readable to git only
    sudo -u git -H chmod o-rwx config/database.yml

### Install Gems

**Note:** As of bundler 1.5.2, you can invoke `bundle install -jN` (where `N` the number of your processor cores) and enjoy the parallel gems installation with measurable difference in completion time (~60% faster). Check the number of your cores with `nproc`. For more information check this [post](http://robots.thoughtbot.com/parallel-gem-installing-using-bundler). First make sure you have bundler >= 1.5.2 (run `bundle -v`) as it addresses some [issues](https://devcenter.heroku.com/changelog-items/411) that were [fixed](https://github.com/bundler/bundler/pull/2817) in 1.5.2.

    # For PostgreSQL (note, the option says "without ... mysql")
    sudo -u git -H bundle install --deployment --without development test mysql aws

    # Or if you use MySQL (note, the option says "without ... postgres")
    sudo -u git -H bundle install --deployment --without development test postgres aws

### Install GitLab Shell

GitLab Shell is an SSH access and repository management software developed specially for GitLab.

    # Run the installation task for gitlab-shell (replace `REDIS_URL` if needed):
    sudo -u git -H bundle exec rake gitlab:shell:install[v2.0.0] REDIS_URL=unix:/var/run/redis/redis.sock RAILS_ENV=production

    # By default, the gitlab-shell config is generated from your main GitLab config.
    # You can review (and modify) the gitlab-shell config as follows:
    sudo -u git -H editor /home/git/gitlab-shell/config.yml

**Note:** If you want to use HTTPS, see [Using HTTPS](#using-https) for the additional steps.

### Initialize Database and Activate Advanced Features

    sudo -u git -H bundle exec rake gitlab:setup RAILS_ENV=production

    # Type 'yes' to create the database tables.

    # When done you see 'Administrator account created:'

**Note:** You can set the Administrator password by supplying it in environmental variable `GITLAB_ROOT_PASSWORD`, eg.:

    sudo -u git -H bundle exec rake gitlab:setup RAILS_ENV=production GITLAB_ROOT_PASSWORD=newpassword

### Install Init Script

Download the init script (will be `/etc/init.d/gitlab`):

    sudo cp lib/support/init.d/gitlab /etc/init.d/gitlab

And if you are installing with a non-default folder or user copy and edit the defaults file:

    sudo cp lib/support/init.d/gitlab.default.example /etc/default/gitlab

If you installed GitLab in another directory or as a user other than the default you should change these settings in `/etc/default/gitlab`. Do not edit `/etc/init.d/gitlab` as it will be changed on upgrade.

Make GitLab start on boot:

    sudo update-rc.d gitlab defaults 21

### Setup Logrotate

    sudo cp lib/support/logrotate/gitlab /etc/logrotate.d/gitlab

### Check Application Status

Check if GitLab and its environment are configured correctly:

    sudo -u git -H bundle exec rake gitlab:env:info RAILS_ENV=production

### Compile Assets

    sudo -u git -H bundle exec rake assets:precompile RAILS_ENV=production

### Start Your GitLab Instance

    sudo service gitlab start
    # or
    sudo /etc/init.d/gitlab restart

## 7. Nginx

**Note:** Nginx is the officially supported web server for GitLab. If you cannot or do not want to use Nginx as your web server, have a look at the [GitLab recipes](https://gitlab.com/gitlab-org/gitlab-recipes/).

### Installation

    sudo apt-get install -y nginx

### Site Configuration

Copy the example site config:

    sudo cp lib/support/nginx/gitlab /etc/nginx/sites-available/gitlab
    sudo ln -s /etc/nginx/sites-available/gitlab /etc/nginx/sites-enabled/gitlab

Make sure to edit the config file to match your setup:

    # Change YOUR_SERVER_FQDN to the fully-qualified
    # domain name of your host serving GitLab.
    sudo editor /etc/nginx/sites-available/gitlab

**Note:** If you want to use HTTPS, replace the `gitlab` Nginx config with `gitlab-ssl`. See [Using HTTPS](#using-https) for HTTPS configuration details.

### Test Configuration

Validate your `gitlab` or `gitlab-ssl` Nginx config file with the following command:

    sudo nginx -t

You should receive `syntax is okay` and `test is successful` messages. If you receive errors check your `gitlab` or `gitlab-ssl` Nginx config file for typos, etc. as indicated in the error message given.

### Restart

    sudo service nginx restart

## Done!

### Double-check Application Status

To make sure you didn't miss anything run a more thorough check with:

    sudo -u git -H bundle exec rake gitlab:check RAILS_ENV=production

If all items are green, then congratulations on successfully installing GitLab!

NOTE: Supply `SANITIZE=true` environment variable to `gitlab:check` to omit project names from the output of the check command.

### Initial Login

Visit YOUR_SERVER in your web browser for your first GitLab login. The setup has created an admin account for you. You can use it to log in:

    root
    5iveL!fe

**Important Note:** Please go over to your profile page and immediately change the password, so nobody can access your GitLab by using this login information later on.

**Enjoy!**

## Advanced Setup Tips

### Using HTTPS

To use GitLab with HTTPS:

1. In `gitlab.yml`:
    1. Set the `port` option in section 1 to `443`.
    1. Set the `https` option in section 1 to `true`.
1. In the `config.yml` of gitlab-shell:
    1. Set `gitlab_url` option to the HTTPS endpoint of GitLab (e.g. `https://git.example.com`).
    1. Set the certificates using either the `ca_file` or `ca_path` option.
1. Use the `gitlab-ssl` Nginx example config instead of the `gitlab` config.
    1. Update `YOUR_SERVER_FQDN`.
    1. Update `ssl_certificate` and `ssl_certificate_key`.
    1. Review the configuration file and consider applying other security and performance enhancing features.

Using a self-signed certificate is discouraged but if you must use it follow the normal directions then:

1. Generate a self-signed SSL certificate:

    ```
    mkdir -p /etc/nginx/ssl/
    cd /etc/nginx/ssl/
    sudo openssl req -newkey rsa:2048 -x509 -nodes -days 3560 -out gitlab.crt -keyout gitlab.key
    sudo chmod o-r gitlab.key
    ```
1. In the `config.yml` of gitlab-shell set `self_signed_cert` to `true`.

### Additional Markup Styles

Apart from the always supported markdown style there are other rich text files that GitLab can display. But you might have to install a dependency to do so. Please see the [github-markup gem readme](https://github.com/gitlabhq/markup#markups) for more information.

### Custom Redis Connection

If you'd like Resque to connect to a Redis server on a non-standard port or on a different host, you can configure its connection string via the `config/resque.yml` file.

    # example
    production: redis://redis.example.tld:6379

If you want to connect the Redis server via socket, then use the "unix:" URL scheme and the path to the Redis socket file in the `config/resque.yml` file.

    # example
    production: unix:/path/to/redis/socket

### Custom SSH Connection

If you are running SSH on a non-standard port, you must change the GitLab user's SSH config.

    # Add to /home/git/.ssh/config
    host localhost          # Give your setup a name (here: override localhost)
        user git            # Your remote git user
        port 2222           # Your port number
        hostname 127.0.0.1; # Your server name or IP

You also need to change the corresponding options (e.g. `ssh_user`, `ssh_host`, `admin_uri`) in the `config\gitlab.yml` file.

### LDAP Authentication

You can configure LDAP authentication in `config/gitlab.yml`. Please restart GitLab after editing this file.

### Using Custom Omniauth Providers

See the [omniauth integration document](doc/integration/omniauth.md)
