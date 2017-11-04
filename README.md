# Installation guide for GitLab 10.1.0 on macOS 10.13.0

## Overview

This is an **unofficial** community-made guide for installing GitLab CE on macOS, based on the official GitLab [installation from source guide](https://docs.gitlab.com/ce/install/installation.html). GitLab does not officially support macOS whatsoever and is largely untested on the platform, so please think twice before installing this for use in a mission-critical environment. That said, GitLab CE does install and run on a macOS server with the following instructions, so if you're looking to set up a GitLab server for personal or small team use, read on!

## Index

The GitLab installation consists of setting up the following components:

1.  Packages / Dependencies
2.  System User
3.  Ruby
4.  Go
5.  Node 
6.  Database
7.  Redis
8.  GitLab
9.  Nginx

## 1. Packages / Dependencies

First, install [Homebrew](https://brew.sh) if you haven't already (this will also automatically install XCode CLI tools if they're not installed yet):

```
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

Then, once it's installed, use it to install the following prerequisite packages:

```
brew install icu4c git logrotate libxml2 cmake pkg-config openssl python re2
```

## 2. System User

To create a user and group for GitLab on your system, you have two choices: through the "Users & Groups" pane in System Preferences, or through the command line via dscl.

To add the `git` user and group through the GUI, create a new Standard user account in System Preferences > Users & Groups with the Full Name 'GitLab' and Account Name 'git' (with a password of your choosing), then create a Group (last option in the New Account dropdown) with the name 'git' and add the user GitLab to it.

Alternatively, to create a new user and group through Terminal, run the following commands in order to create the group and user `git`:

```bash
LastUserID=$(dscl . -list /Users UniqueID | awk '{print $2}' | sort -n | tail -1)
NextUserID=$((LastUserID + 1))
sudo dscl . create /Users/git
sudo dscl . create /Users/git RealName "GitLab"
sudo dscl . create /Users/git hint "Password Hint"
sudo dscl . create /Users/git UniqueID $NextUserID
LastGroupID=$(dscl . readall /Groups | grep PrimaryGroupID | awk '{ print $2 }' | sort -n | tail -1)
NextGroupID=$(($LastGroupID + 1 ))
sudo dscl . create /Groups/git
sudo dscl . create /Groups/git RealName "GitLab"
sudo dscl . create /Groups/git passwd "*"
sudo dscl . create /Groups/git gid $NextGroupID
sudo dscl . create /Users/git PrimaryGroupID $NextGroupID
sudo dscl . create /Users/git UserShell $(which bash)
sudo dscl . create /Users/git NFSHomeDirectory /Users/git
sudo cp -R /System/Library/User\ Template/English.lproj /Users/git
```

Regardless of how you create your git user, you will need to proper ownership of its home directory using `sudo chown -R git:git /Users/git`.

To hide the git user from the login screen:

```
sudo defaults write /Library/Preferences/com.apple.loginwindow HiddenUsersList -array-add git
```

To unhide it:

```
sudo defaults delete /Library/Preferences/com.apple.loginwindow HiddenUsersList
```

## 3. Ruby

> The use of Ruby version managers such as [RVM](http://rvm.io/), [rbenv](https://github.com/sstephenson/rbenv) or [chruby](https://github.com/postmodern/chruby) with GitLab in production frequently leads to hard to diagnose problems. For example, GitLab Shell is called from OpenSSH and having a version manager can prevent pushing and pulling over SSH. Version managers are not supported and we strongly advise everyone to follow the instructions below to use a system Ruby.

macOS High Sierra ships with ruby 2.3.3, so there is no longer any need to install a brewed version. All you need to do is install the bundler gem:

```
sudo gem install bundler --no-ri --no-rdoc
```


## 4. Go

Since GitLab 8.0, GitLab has several daemons written in Go. To install GitLab we need a Go compiler.

```
brew install go
```

## 5. Node

Since GitLab 8.17, GitLab requires the use of node >= v4.3.0 to compile javascript assets, and yarn >= v0.17.0 to manage javascript dependencies.

```
brew install node yarn
```

## 6. Database

Gitlab recommends using a PostgreSQL database. But you can use MySQL too, see [MySQL setup guide](https://docs.gitlab.com/ce/install/database_mysql.html).

```
brew install postgresql
brew services start postgresql
```

1. Login to PostgreSQL

    ```
    psql -d postgres
    ```

2. Create a user for GitLab.

    ```sql
    CREATE USER git CREATEDB;
    ALTER USER git WITH ENCRYPTED PASSWORD 'MY_SECRET_PASSWORD';
    ```

3. Create the pg_trgm extension (required for GitLab 8.6+):

    ```sql
    CREATE EXTENSION IF NOT EXISTS pg_trgm;
    ```

4. Create the GitLab production database & grant all privileges on database, then quit the database session

    ```sql
    CREATE DATABASE gitlabhq_production OWNER git;
    \q
    ```

5. Try connecting to the new database with the new user

    ```
    sudo -u git -H psql -d gitlabhq_production
    ```

6. Check if the `pg_trgm` extension is enabled:

    ```sql
    SELECT true AS enabled
    FROM pg_available_extensions
    WHERE name = 'pg_trgm'
    AND installed_version IS NOT NULL;
    ```

    If the extension is enabled this will produce the following output:

    ```
    enabled
    ---------
     t
    (1 row)
    ```


## 7. Redis

GitLab also requires the redis package, which can also be installed with Homebrew:

```
brew install redis
```

Redis config is located in `/usr/local/etc/redis.conf`. Make a copy:

```
cp /usr/local/etc/redis.conf /usr/local/etc/redis.conf.orig
```

Disable Redis listening on TCP by setting 'port' to 0

```
sed 's/^port .*/port 0/' /usr/local/etc/redis.conf.orig | sudo tee /usr/local/etc/redis.conf
```

Edit file (`nano /usr/local/etc/redis.conf`) and uncomment:

```
unixsocket /tmp/redis.sock
unixsocketperm 777
```

Then, you can start redis as a service that will persist across reboots by running

```
brew services start redis
```

## 8. GitLab

```
cd /Users/git
```

### Clone the Source

Clone GitLab repository

```
sudo -u git -H git clone https://gitlab.com/gitlab-org/gitlab-ce.git -b 10-1-stable gitlab
```

**Note:** You can change `10-1-stable` to `master` if you want the *bleeding edge* version, but never install master on a production server!

### Configure It

```bash
# Go to GitLab installation folder
cd /Users/git/gitlab

# Copy the example GitLab config and edit it to suit macOS's directory structure
sudo -u git -H cp config/gitlab.yml.example config/gitlab.yml
sudo -u git sed -i "" "s/\/usr\/bin\/git/\/usr\/local\/bin\/git/g" config/gitlab.yml
sudo -u git sed -i "" "s/\/home/\/Users/g" config/gitlab.yml

# Update GitLab config file, follow the directions at top of file
sudo -u git -H nano config/gitlab.yml

# Copy the example secrets file
sudo -u git -H cp config/secrets.yml.example config/secrets.yml
sudo -u git -H chmod 0600 config/secrets.yml

# Make sure GitLab can write to the log/ and tmp/ directories
sudo chown -R git log/
sudo chown -R git tmp/
sudo chmod -R u+rwX,go-w log/
sudo chmod -R u+rwX tmp/

# Make sure GitLab can write to the tmp/pids/ and tmp/sockets/ directories
sudo chmod -R u+rwX tmp/pids/
sudo chmod -R u+rwX tmp/sockets/

# Create the public/uploads/ directory
sudo -u git -H mkdir public/uploads/

# Make sure only the GitLab user has access to the public/uploads/ directory
# now that files in public/uploads are served by gitlab-workhorse
sudo chmod 0700 public/uploads

# Change the permissions of the directory where CI job traces are stored
sudo chmod -R u+rwX builds/

# Change the permissions of the directory where CI artifacts are stored
sudo chmod -R u+rwX shared/artifacts/

# Change the permissions of the directory where GitLab Pages are stored
sudo chmod -R u+rwX shared/pages

# Copy the example Unicorn config
sudo -u git -H cp config/unicorn.rb.example config/unicorn.rb
sudo -u git sed -i "" "s/\/home/\/Users/g" config/unicorn.rb

# Find number of cores
sysctl -n hw.ncpu

# Enable cluster mode if you expect to have a high load instance
# Ex. change amount of workers to 3 for 2GB RAM server
# Set the number of workers to at least the number of cores
sudo -u git -H nano config/unicorn.rb

# Copy the example Rack attack config
sudo -u git -H cp config/initializers/rack_attack.rb.example config/initializers/rack_attack.rb

# Configure Git global settings for git user, used when editing via web editor
sudo -u git -H git config --global core.autocrlf input

# Disable 'git gc --auto' because GitLab runs 'git gc' for us already.
sudo -u git -H git config --global gc.auto 0

# Configure Git to generate packfile bitmaps (introduced in Git 2.0) on the GitLab server during git gc.
sudo -u git -H git config --global repack.writeBitmaps true

# Configure Redis connection settings
sudo -u git -H cp config/resque.yml.example config/resque.yml

# Change the Redis socket path to '/tmp/redis.sock'
sudo -u git -H nano config/resque.yml

```

**Important Note:** Make sure to edit both `gitlab.yml` and `unicorn.rb` to match your setup.

**Note:** If you want to use HTTPS, see [Using HTTPS](#using-https) for the additional steps.

### Configure GitLab DB Settings

PostgreSQL only:

```
sudo -u git cp config/database.yml.postgresql config/database.yml
```

MySQL only:

```
sudo -u git cp config/database.yml.mysql config/database.yml
```

MySQL and remote PostgreSQL only:
Update username/password in config/database.yml.
You only need to adapt the production settings (first part).
If you followed the database guide then please do as follows:
Change 'secure password' with the value you have given to $password
You can keep the double quotes around the password

```
sudo -u git -H nano config/database.yml
```

PostgreSQL and MySQL:
Make config/database.yml readable to git only

```
sudo -u git -H chmod o-rwx config/database.yml
```

### Install Gems

**Note:** As of bundler 1.5.2, you can invoke `bundle install -jN` (where `N` the number of your processor cores) and enjoy the parallel gems installation with measurable difference in completion time (~60% faster). Check the number of your cores with `nproc`. For more information check this [post](http://robots.thoughtbot.com/parallel-gem-installing-using-bundler). First make sure you have bundler >= 1.5.2 (run `bundle -v`) as it addresses some [issues](https://devcenter.heroku.com/changelog-items/411) that were [fixed](https://github.com/bundler/bundler/pull/2817) in 1.5.2.

Preparation:

```
sudo su git
cd ~/gitlab/
```

First, we need to set some build flags for the charlock_holmes gem in order for it to compile properly on macOS:

```
bundle config build.charlock_holmes --with-icu-dir=/usr/local/opt/icu4c --with-cxxflags=-std=c++11
```

Then we can install all GitLab's required gems using bundler. Note that we have to preface the install commands with `env ARCHFLAGS="-arch x86_64 -Wno-error=reserved-user-defined-literal"` for things to work properly; this is because High Sierra's Ruby.h C header conflicts with the warning-to-error settings of the default Clang compiler and prevents some gems with native extensions (e.g. `charlock_holmes` and `re2`) from building without those environment variables set.

For PostgreSQL (note, the option says "without ... mysql")

```
env ARCHFLAGS="-arch x86_64 -Wno-error=reserved-user-defined-literal" bundle install --deployment --without development test mysql aws kerberos
```

Or if you use MySQL (note, the option says "without ... postgres")

```
env ARCHFLAGS="-arch x86_64 -Wno-error=reserved-user-defined-literal" bundle install --deployment --without development test postgres aws kerberos
```



**Note:** If you want to use Kerberos for user authentication, then omit `kerberos` in the `--without` option above.

### Install GitLab Shell

GitLab Shell is an SSH access and repository management software developed specially for GitLab.

Run the installation task for gitlab-shell (replace `REDIS_URL` if needed):

```
sudo -u git -H bundle exec rake gitlab:shell:install REDIS_URL=unix:/tmp/redis.sock RAILS_ENV=production
```

By default, the gitlab-shell config is generated from your main GitLab config.
You can review (and modify) the gitlab-shell config as follows:

```
sudo -u git -H nano /Users/git/gitlab-shell/config.yml
```

**Note:** If you want to use HTTPS, see [Using HTTPS](#using-https) for the additional steps.

**Note:** Make sure your hostname can be resolved on the machine itself by either a proper DNS record or an additional line in /etc/hosts ("127.0.0.1  hostname"). This might be necessary for example if you set up gitlab behind a reverse proxy. If the hostname cannot be resolved, the final installation check will fail with "Check GitLab API access: FAILED. code: 401" and pushing commits will be rejected with "[remote rejected] master -> master (hook declined)".

### Install gitlab-workhorse

```
sudo -u git -H bundle exec rake "gitlab:workhorse:install[/Users/git/gitlab-workhorse]" RAILS_ENV=production
```

### Initialize Database and Activate Advanced Features

```
sudo -u git -H bundle exec rake gitlab:setup RAILS_ENV=production
```

Type 'yes' to create the database tables.
When done you see 'Administrator account created:

**Note:** You can set the Administrator/root password by supplying it in environmental variable `GITLAB_ROOT_PASSWORD` as seen below. If you don't set the password (and it is set to the default one) please wait with exposing GitLab to the public internet until the installation is done and you've logged into the server the first time. During the first login you'll be forced to change the default password.

```
sudo -u git -H bundle exec rake gitlab:setup RAILS_ENV=production GITLAB_ROOT_PASSWORD=yourpassword
```

### Secure secrets.yml

The `secrets.yml` file stores encryption keys for sessions and secure variables.
Backup `secrets.yml` someplace safe, but don't store it in the same place as your database backups.
Otherwise your secrets are exposed if one of your backups is compromised.

### Install Init Script

Download the init script (will be `/etc/init.d/gitlab`):

```
cd /Users/git/gitlab
sudo mkdir -p /etc/init.d/
sudo mkdir -p /etc/default/
sudo cp lib/support/init.d/gitlab /etc/init.d/gitlab
```

Since we are installing to a folder other than default, we need to edit the defaults file after copying:

```
sudo cp lib/support/init.d/gitlab.default.example /etc/default/gitlab
sudo sed -i "" "s/\/home/\/Users/g" /etc/default/gitlab
```

If you installed GitLab in another directory or as a user other than the default you should change these settings in `/etc/default/gitlab`. Do not edit `/etc/init.d/gitlab` as it will be changed on upgrade.


### Install Gitaly

    # Fetch Gitaly source with Git and compile with Go
    sudo -u git -H bundle exec rake "gitlab:gitaly:install[/Users/git/gitaly]" RAILS_ENV=production

You can specify a different Git repository by providing it as an extra paramter:

    sudo -u git -H bundle exec rake "gitlab:gitaly:install[/Users/git/gitaly,https://example.com/gitaly.git]" RAILS_ENV=production

Next, make sure gitaly configured:

    # Restrict Gitaly socket access
    sudo chmod 0700 /Users/git/gitlab/tmp/sockets/private
    sudo chown git /Users/git/gitlab/tmp/sockets/private

    # If you are using non-default settings you need to update config.toml
    cd /Users/git/gitaly
    sudo -u git -H editor config.toml

For more information about configuring Gitaly see
[doc/administration/gitaly](https://docs.gitlab.com/ce/administration/gitaly/).

### Setup Logrotate
```
sudo cp lib/support/logrotate/gitlab /usr/local/etc/logrotate.d/gitlab
sudo sed -i "" "s/\/home/\/Users/g" /usr/local/etc/logrotate.d/gitlab
brew services start logrotate
```

### Check Application Status

Check if GitLab and its environment are configured correctly:
```
sudo -u git -H bundle exec rake gitlab:env:info RAILS_ENV=production
```

### Compile GetText PO files

```
sudo -u git -H bundle exec rake gettext:pack RAILS_ENV=production
sudo -u git -H bundle exec rake gettext:po_to_json RAILS_ENV=production
```

### Compile Assets
```
sudo -u git -H yarn install --production --pure-lockfile
sudo -u git -H bundle exec rake assets:precompile RAILS_ENV=production NODE_ENV=production
```

### Start Your GitLab Instance

Before we can start our GitLab instance, we first have to edit a file to prevent Unicorn from crashing on launch on High Sierra (see [this](https://blog.phusion.nl/2017/10/13/why-ruby-app-servers-break-on-macos-high-sierra-and-what-can-be-done-about-it/ for more details)):

```
cd /Users/git/gitlab/
sudo -u git -H nano bin/web
```
then, add the line

```
export DYLD_INSERT_LIBRARIES=/System/Library/Frameworks/Foundation.framework/Versions/Current/Foundation
```

between the `#/bin/sh` line at the top and the `cd $(dirname $0)/..` line, and save the file. Once this has been done, you can start your gitlab instance by running
```
sudo -u git -H /etc/init.d/gitlab start
```

## 9. Nginx

**Note:** Nginx is the officially supported web server for GitLab. If you cannot or do not want to use Nginx as your web server, have a look at the [GitLab recipes](https://gitlab.com/gitlab-org/gitlab-recipes/).

### Installation
```
brew tap homebrew/nginx
brew install nginx-full --with-realip
sudo mkdir -p /var/log/nginx/
```

### Site Configuration

Default nginx configuration has an example server on port 8080, same as Gitlab Unicorn instance, which will collide and Gitlab won't start.
Edit nginx configuration and comment out whole example server block for it to work together:

```
sudo nano /usr/local/etc/nginx/nginx.conf
```

Copy the example site config:

```
sudo cp lib/support/nginx/gitlab /usr/local/etc/nginx/servers/gitlab
sudo sed -i "" "s/\/home/\/Users/g" /usr/local/etc/nginx/servers/gitlab
```

Make sure to edit the config file to match your setup:

Change YOUR_SERVER_FQDN to the fully-qualified domain name of your host serving GitLab.

```
sudo nano /usr/local/etc/nginx/servers/gitlab
```

**Note:** If you want to use HTTPS, replace the `gitlab` Nginx config with `gitlab-ssl`. See [Using HTTPS](#using-https) for HTTPS configuration details.

### Test Configuration

Validate your `gitlab` or `gitlab-ssl` Nginx config file with the following command:

```
sudo nginx -t
```

You should receive `syntax is okay` and `test is successful` messages. If you receive errors check your `gitlab` or `gitlab-ssl` Nginx config file for typos, etc. as indicated in the error message given.

### Start

```
sudo brew services start nginx-full
```

## Done!

### Double-check Application Status

To make sure you didn't miss anything run a more thorough check with:

```
sudo -u git -H bundle exec rake gitlab:check RAILS_ENV=production
```

If all items are green, then congratulations on successfully installing GitLab!

NOTE: Supply `SANITIZE=true` environment variable to `gitlab:check` to omit project names from the output of the check command.

### Initial Login

Visit YOUR_SERVER in your web browser for your first GitLab login. The setup has created a default admin account for you. You can use it to log in:

```
root
5iveL!fe
```

**Important Note:** On login you'll be prompted to change the password.

**Enjoy!**

You can use `sudo sh /etc/init.d/gitlab start`, `sudo sh /etc/init.d/gitlab stop` and `sudo sh /etc/init.d/gitlab restart` to manually start, stop and restart GitLab.

### Autostart on boot

Copy Nginx and Gitlab plists and load it:

```
sudo cp /usr/local/opt/nginx/homebrew.mxcl.nginx.plist /Library/LaunchDaemons/
sudo cp com.webentity.gitlab.plist /Library/LaunchDaemons/
sudo launchctl load /Library/LaunchDaemons/homebrew.mxcl.nginx.plist
sudo launchctl load /Library/LaunchDaemons/com.webentity.gitlab.plist
```

## Advanced Setup Tips

### Automated backups

Create log directory, copy in backup plist and load it
To enable backup to function you will need to configure the backup options in `config/gitlab.yml`
```
sudo mkdir -p /usr/local/var/log/gitlab
sudo chown git:git /usr/local/var/log/gitlab
sudo cp com.gitlab.backup.plist /Library/LaunchDaemons/
sudo launchctl load /Library/LaunchDaemons/com.gitlab.backup.plist
```

Example external HD backup config settings
```
  ## Backup settings
  backup:
    path: "tmp/backups"   # Relative paths are relative to Rails.root (default: tmp/backups/)
    # archive_permissions: 0640 # Permissions for the resulting backup.tar file (default: 0600)
    # keep_time: 604800   # default: 0 (forever) (in seconds)
    # pg_schema: public     # default: nil, it means that all schemas will be backed up
    upload:
    #   # Fog storage connection settings, see http://fog.io/storage/ .
      connection:
        provider: Local
        local_root: '/Volumes/BackupHD/gitlab_backups'
      remote_directory: '.'
    #     provider: AWS
    #     region: eu-west-1
    #     aws_access_key_id: AKIAKIAKI
    #     aws_secret_access_key: 'secret123'
    #   # The remote 'directory' to store your backups. For S3, this would be the bucket name.
    #   remote_directory: 'my.s3.bucket'
    #   # Use multipart uploads when file size reaches 100MB, see
    #   #  http://docs.aws.amazon.com/AmazonS3/latest/dev/uploadobjusingmpu.html
    #   multipart_chunk_size: 104857600
    #   # Turns on AWS Server-Side Encryption with Amazon S3-Managed Keys for backups, this is optional
    #   # encryption: 'AES256'
```

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

### SMTP configuration

If you're installing from source and use SMTP to deliver mail, you will need to add the following line to config/initializers/smtp_settings.rb:

```
ActionMailer::Base.delivery_method = :smtp
```

See [smtp_settings.rb.sample](https://gitlab.com/gitlab-org/gitlab-ce/blob/10-1-stable/config/initializers/smtp_settings.rb.sample#L13) as an example.

### More

You can find more tips in [official documentation](https://docs.gitlab.com/ce/install/installation.html#advanced-setup-tips).
