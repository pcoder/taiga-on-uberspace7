# How to install taiga on uberspace 7


## Steps:
---------

### 1. Config and parameters

Make sure to have a good configuration machine and prepare a list of parameters that will be frequently used during installation.
As suggested by taiga these are :

```
    IP: 80.88.23.45

    Hostname: example.com (which points to 80.88.23.45)

    Username: taiga

    System memory: >=1GB (needed for compilation of lxml)

    Working directory: /home/taiga/ (default for user taiga)
```

### 2. Install postgres and create database

2.1 Download and extract the source code
```
[taiga@dysnomia ~]$ mkdir ~/postgres/
[taiga@dysnomia ~]$ cd ~/postgres/
[taiga@dysnomia ~]$ curl -O https://download.postgresql.org/pub/source/v9.6.10/postgresql-9.6.10.tar.gz
[taiga@dysnomia ~]$ tar -xvzf ~/postgres/postgresql-9.6.10.tar.gz
[taiga@dysnomia ~]$
```

2.2 Source Code Configuration, Compiling and Installation

```
[taiga@dysnomia ~]$ cd ~/postgres/postgresql-9.6.10
[taiga@dysnomia ~]$ ./configure --prefix=$HOME/opt/postgresql/ --with-python PYTHON=/usr/bin/python3.6 --without-readline
[taiga@dysnomia ~]$ make world
[taiga@dysnomia ~]$ make install-world
[taiga@dysnomia ~]$
```
2.3 Update `~/.bashrc` so that postgres tools are in the path. Append the following code to `~/.bashrc`

```
export HOME=/home/pcoder1
export PATH=$HOME/opt/postgresql/bin/:$PATH
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$HOME/opt/postgresql/lib
export PGPASSFILE=$HOME/.pgpass
```

```
[taiga@dysnomia ~]$ source ~/.bashrc
[taiga@dysnomia ~]$ psql --version
psql (PostgreSQL) 9.6.10
```

2.4 Initialize database

2.4.1 Create the file `~/.pgpass` with the following content. Change the username and passowrd fields as per your liking
```
#hostname:port:database:username:password (min 64 characters)
*:*:*:isabell:1234567890123456789012345678901234567890123456789012345678901234
```

2.4.2

```
 [taiga@dysnomia ~]$ chmod 0600 ~/.pgpass
 [taiga@dysnomia ~]$ cp ~/.pgpass ~/pgpass.temp
 [taiga@dysnomia ~]$ cat ~/pgpass.temp
```
2.4.3 Edit the pgpass.temp file to contain only the password

```
[taiga@dysnomia ~]$ cat ~/pgpass.temp
1234567890123456789012345678901234567890123456789012345678901234

```
2.4.4 Initialize DB

```
[taiga@dysnomia ~]$ initdb --pwfile="/home/taiga/pgpass.temp" --auth=md5 -E UTF8 -D ~/opt/postgresql/data/


The files belonging to this database system will be owned by user "taiga".
This user must also own the server process.

The database cluster will be initialized with locale "en_GB.UTF-8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

creating directory /home/taiga/opt/postgresql/data ... ok
creating subdirectories ... ok
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting dynamic shared memory implementation ... posix
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok

Success. You can now start the database server using:

    pg_ctl -D /home/taiga/opt/postgresql/data/ -l logfile start
```

2.4.5 Remove unused `~/pgpass.temp`

```
 [taiga@dysnomia ~]$ rm ~/pgpass.temp
```

2.4.6 Obtain a free port to be used for DB port number

```
[taiga@dysnomia ~]$ FREEPORT=$(( $RANDOM % 4535 + 61000 )); ss -ln src :$FREEPORT | grep $FREEPORT && echo "try again" || echo $FREEPORT
```
63921 => this is the DB port that I took for the database

2.4.7 Add the following to ~/.bashrc

```
 export PGHOST=localhost
 export PGPORT=63921
```

2.4.8  Edit ``~/opt/postgresql/data/postgresql.conf`` and set the key values ``listen_adresses``, ``port`` and ``unix_socket_directories``:

.. warning:: Replace the port number with the one you wrote down earlier and replace ``<username>`` with your username!

.. code-block:: postgres
 :emphasize-lines: 7,11,14

 #------------------------------------------------------------------------------
 # CONNECTIONS AND AUTHENTICATION
 #------------------------------------------------------------------------------

 # - Connection Settings -

 listen_addresses = 'localhost'         # what IP address(es) to listen on;
                                        # comma-separated list of addresses;
                                        # defaults to 'localhost'; use '*' for all
                                        # (change requires restart)
 port = 63921                            # (change requires restart)
 max_connections = 100                  # (change requires restart)
 #superuser_reserved_connections = 3    # (change requires restart)
 unix_socket_directories = '/home/<username>/tmp'      # comma-separated list of directories
                                        # (change requires restart)
 #unix_socket_group = ''                # (change requires restart)
 #unix_socket_permissions = 0777        # begin with 0 to use octal notation
                                        # (change requires restart)
 #bonjour = off                         # advertise server via Bonjour
                                        # (change requires restart)
 #bonjour_name = ''                     # defaults to the computer name
                                        # (change requires restart)


2.4.9 Setup service

Create ``~/etc/services.d/postgresql.ini`` with the following content:

.. warning:: Replace ``<username>`` with your username!

.. code-block:: ini

```
[program:postgresql]
command=/home/taiga/opt/postgresql/bin/postgres -D /home/taiga/opt/postgresql/data/
autostart=yes
autorestart=yes

```

2.4.10 Update supervisor

::

[taiga@dysnomia ~]$ supervisorctl reread
postgresql: available
[taiga@dysnomia ~]$
[taiga@dysnomia ~]$ supervisorctl update
[taiga@dysnomia ~]$
[taiga@dysnomia ~]$ supervisorctl status
postgresql                       RUNNING   pid 15477, uptime 0:00:07
[taiga@dysnomia ~]$

2.4.11 Create user and datbase

::

[taiga@dysnomia ~]$ createuser -h localhost -p 63921 taiga
[taiga@dysnomia ~]$ createdb -h localhost -p 63921  taiga -O taiga --encoding='utf-8' --locale=en_US.utf8 --template=template0


### 3. Setup python and Django

why pip3.6 ?? Because we've use /usr/bin/python3 during installation of postgres and it refers to python3.6

3.1 Create virtualenv and install requirements

::

[taiga@dysnomia ~]$ pip3.6 install virtualenvwrapper --user
[taiga@dysnomia ~]$ source ~/.local/bin/virtualenvwrapper_lazy.sh
[taiga@dysnomia ~]$ mkvirtualenv -p /usr/bin/python3.6 taiga


.. code-block:: console

[taiga@dysnomia ~]$ git clone https://github.com/taigaio/taiga-back.git taiga-back
[taiga@dysnomia ~]$ cd taiga-back
[taiga@dysnomia taiga-back]$ git checkout stable
[taiga@dysnomia taiga-back]$ pip3.6 install -r requirements.txt

.. code-block:: bash
export PYTHONPATH=$HOME/.virtualenvs/taiga/lib/python3.6/site-packages

3.2 Do database migration

.. code-block:: bash
(taiga) [taiga@dysnomia taiga-back]$ python manage.py migrate --noinput
(taiga) [taiga@dysnomia taiga-back]$ python manage.py loaddata initial_user
(taiga) [taiga@dysnomia taiga-back]$ python manage.py loaddata initial_project_templates
(taiga) [taiga@dysnomia taiga-back]$ python manage.py loaddata initial_role
(taiga) [taiga@dysnomia taiga-back]$ python manage.py collectstatic --noinput


Ignore this error:
```
(taiga) [taiga@dysnomia taiga-back]$ python manage.py loaddata initial_role
Trying import local.py settings...
Trying import development.py settings...
CommandError: No fixture named 'initial_role' found.
```
3.3 Test if Django installation works

```
(taiga) [taiga@dysnomia taiga-back]$ python manage.py runserver
```



### 4 Install taiga-front

4.1 Use ruby version 2.7 available on uberspace

```
(taiga) [taiga@dysnomia taiga-back]$ uberspace tools version list ruby
- 2.5
- 2.6
- 2.7

(taiga) [taiga@dysnomia taiga-back]$ uberspace tools version show ruby
Using 'Ruby' version: '2.7'
```

4.2 Install sass and scss_lint

```
(taiga) [taiga@dysnomia taiga-back]$ gem install --user-install sass scss_lint
WARNING:  You don't have /home/taiga/.gem/ruby/2.7.0/bin in your PATH,
	  gem executables will not run.

Ruby Sass has reached end-of-life and should no longer be used.

* If you use Sass as a command-line tool, we recommend using Dart Sass, the new
  primary implementation: https://sass-lang.com/install

* If you use Sass as a plug-in for a Ruby web framework, we recommend using the
  sassc gem: https://github.com/sass/sassc-ruby#readme

* For more details, please refer to the Sass blog:
  https://sass-lang.com/blog/posts/7828841

Successfully installed sass-3.7.4
Successfully installed scss_lint-0.59.0
2 gems installed

```

4.3 Create `~/.npmrc`

```
cat > ~/.npmrc <<__EOF__
prefix=$HOME
umask=077
__EOF__
```

4.4 Install gulp and bower
```
(taiga) [taiga@dysnomia taiga-back]$ npm install -g --prefix=$HOME gulp bower
npm WARN deprecated bower@1.8.8: We don't recommend using Bower for new projects. Please consider Yarn and Webpack or Parcel. You can read how to migrate legacy project here: https://bower.io/blog/2017/how-to-migrate-away-from-bower/
npm WARN deprecated chokidar@2.1.8: Chokidar 2 will break on node v14+. Upgrade to chokidar 3 with 15x less dependencies.
npm WARN deprecated urix@0.1.0: Please see https://github.com/lydell/urix#deprecated
npm WARN deprecated resolve-url@0.2.1: https://github.com/lydell/resolve-url#deprecated
npm WARN deprecated fsevents@1.2.13: fsevents 1 will break on node v14+ and could be using insecure binaries. Upgrade to fsevents 2.
debug2: channel 0: window 999271 sent adjust 49305ill extract async-done@1.3.2
/home/taiga/bin/bower -> /home/taiga/lib/node_modules/bower/bin/bower
/home/taiga/bin/gulp -> /home/taiga/lib/node_modules/gulp/bin/gulp.js
npm WARN optional SKIPPING OPTIONAL DEPENDENCY: fsevents@^1.2.7 (node_modules/gulp/node_modules/chokidar/node_modules/fsevents):
npm WARN notsup SKIPPING OPTIONAL DEPENDENCY: Unsupported platform for fsevents@1.2.13: wanted {"os":"darwin","arch":"any"} (current: {"os":"linux","arch":"x64"})

+ gulp@4.0.2
+ bower@1.8.8
added 332 packages from 222 contributors in 18.452s
```

4.5 Download and extract taiga-front

```
(taiga) [taiga@dysnomia mrr]$ cd ~/mrr
(taiga) [taiga@dysnomia mrr]$ git clone https://github.com/taigaio/taiga-front.git taiga-front
Cloning into 'taiga-front'...
remote: Enumerating objects: 223, done.
remote: Counting objects: 100% (223/223), done.
remote: Compressing objects: 100% (154/154), done.
remote: Total 71380 (delta 113), reused 124 (delta 65), pack-reused 71157
Receiving objects: 100% (71380/71380), 41.55 MiB | 13.24 MiB/s, done.
Resolving deltas: 100% (51098/51098), done.
(taiga) [taiga@dysnomia mrr]$  cd taiga-front/
(taiga) [taiga@dysnomia taiga-front]$ git checkout stable
Branch 'stable' set up to track remote branch 'stable' from 'origin'.
Switched to a new branch 'stable'
```

4.6 Install node modules

```
(taiga) [taiga@dysnomia taiga-front]$ npm install
debug2: channel 0: window 999396 sent adjust 49180ball no local data for node-s
debug2: channel 0: window 999308 sent adjust 49268finalize /home/taiga/mrr/taig
npm WARN lifecycle The node binary used for scripts is /usr/bin/node but npm is using /opt/nodejs12/bin/node itself. Use the `--scripts-prepend-node-path` option to include the path for the node binary npm was executed with.

> node-sass@4.14.1 install /home/taiga/mrr/taiga-front/node_modules/node-sass
> node scripts/install.js

Downloading binary from https://github.com/sass/node-sass/releases/download/v4.14.1/linux-x64-72_binding.node
Download complete .] - :
Binary saved to /home/taiga/mrr/taiga-front/node_modules/node-sass/vendor/linux-x64-72/binding.node
Caching binary to /home/taiga/.npm/node-sass/4.14.1/linux-x64-72_binding.node

> pre-commit@1.2.2 install /home/taiga/mrr/taiga-front/node_modules/pre-commit
> node install.js


> core-js@2.6.11 postinstall /home/taiga/mrr/taiga-front/node_modules/core-js
> node -e "try{require('./postinstall')}catch(e){}"

Thank you for using core-js ( https://github.com/zloirock/core-js ) for polyfilling JavaScript standard library!

The project needs your help! Please consider supporting of core-js on Open Collective or Patreon: 
> https://opencollective.com/core-js 
> https://www.patreon.com/zloirock 

Also, the author of core-js ( https://github.com/zloirock ) is looking for a good job -)


> core-js@3.6.5 postinstall /home/taiga/mrr/taiga-front/node_modules/gulp-cache/node_modules/core-js
> node -e "try{require('./postinstall')}catch(e){}"


> spawn-sync@1.0.15 postinstall /home/taiga/mrr/taiga-front/node_modules/spawn-sync
> node postinstall


> node-sass@4.14.1 postinstall /home/taiga/mrr/taiga-front/node_modules/node-sass
> node scripts/build.js

Binary found at /home/taiga/mrr/taiga-front/node_modules/node-sass/vendor/linux-x64-72/binding.node
Testing binary
Binary is fine

> gifsicle@5.1.0 postinstall /home/taiga/mrr/taiga-front/node_modules/gifsicle
> node lib/install.js

  ⚠ Response code 404 (Not Found)
  ⚠ gifsicle pre-build test failed
  ℹ compiling from source
  ✔ gifsicle built successfully

> mozjpeg@6.0.1 postinstall /home/taiga/mrr/taiga-front/node_modules/mozjpeg
> node lib/install.js

  ✔ mozjpeg pre-build test passed successfully

> optipng-bin@6.0.0 postinstall /home/taiga/mrr/taiga-front/node_modules/optipng-bin
> node lib/install.js

  ✔ optipng pre-build test passed successfully
npm WARN The package bluebird is included as both a dev and production dependency.
npm WARN optional SKIPPING OPTIONAL DEPENDENCY: fsevents@1.2.13 (node_modules/fsevents):
npm WARN notsup SKIPPING OPTIONAL DEPENDENCY: Unsupported platform for fsevents@1.2.13: wanted {"os":"darwin","arch":"any"} (current: {"os":"linux","arch":"x64"})

added 2006 packages from 1539 contributors and audited 2022 packages in 72.767s

89 packages are looking for funding
  run `npm fund` for details

found 75 vulnerabilities (30 low, 4 moderate, 36 high, 5 critical)
  run `npm audit fix` to fix them, or `npm audit` for details
```

4.7 Install bower (it did not work out for me)

```
(taiga) [taiga@dysnomia taiga-front]$ bower install
bower ENOENT        No bower.json present
```

4.8 Create taiga-front config

```
(taiga) [taiga@dysnomia taiga-front]$ cp conf/conf.example.json conf/main.json
```


Put the following in ~/mrr/taiga-front/conf/main.json
```
{
    "api": "https://taiga.uber.space/api/v1/",
    "eventsUrl": null,
    "eventsMaxMissedHeartbeats": 5,
    "eventsHeartbeatIntervalTime": 60000,
    "eventsReconnectTryInterval": 10000,
    "debug": true,
    "debugInfo": false,
    "defaultLanguage": "en",
    "themes": ["taiga"],
    "defaultTheme": "taiga",
    "defaultLoginEnabled": true,
    "publicRegisterEnabled": true,
    "feedbackEnabled": true,
    "supportUrl": "https://tree.taiga.io/support/",
    "privacyPolicyUrl": null,
    "termsOfServiceUrl": null,
    "GDPRUrl": null,
    "maxUploadFileSize": null,
    "contribPlugins": [],
    "tagManager": { "accountId": null },
    "tribeHost": null,
    "importers": [],
    "gravatar": false,
    "rtlLanguages": ["fa"]
}
```

4.9 Build the front-end

```
(taiga) [taiga@dysnomia taiga-front]$ gulp deploy
[21:09:43] Using gulpfile ~/mrr/taiga-front/gulpfile.js
[21:09:43] Starting 'deploy'...
[21:09:43] Starting 'clear'...
[21:09:43] Starting 'clear-sass-cache'...
[21:09:43] Finished 'clear-sass-cache' after 747 μs
[21:09:43] Starting '<anonymous>'...
[21:09:43] Finished '<anonymous>' after 1.16 ms
[21:09:43] Finished 'clear' after 3.34 ms
[21:09:43] Starting 'delete-old-version'...
[21:09:43] Finished 'delete-old-version' after 3.66 ms
[21:09:43] Starting 'delete-tmp'...
[21:09:43] Finished 'delete-tmp' after 1.27 ms
[21:09:43] Starting 'copy'...
[21:09:43] Starting 'jade-deploy'...
[21:09:43] Starting 'app-deploy'...
[21:09:43] Starting 'jslibs-deploy'...
[21:09:43] Starting 'link-images'...
[21:09:43] Starting 'compile-themes'...
[21:09:43] Starting 'copy-fonts'...
[21:09:43] Starting 'copy-theme-fonts'...
[21:09:43] Starting 'copy-images'...
[21:09:43] Starting 'copy-emojis'...
[21:09:43] Starting 'copy-prism'...

.
.
.
```

4.10 Copy the frontend files to html folder
```
(taiga) [taiga@dysnomia mrr]$ cp -rf taiga-front/dist/* /var/www/virtual/taiga/html/
(taiga) [taiga@dysnomia mrr]$ cp -rf taiga-back/static /var/www/virtual/taiga/html/
(taiga) [taiga@dysnomia mrr]$ cp -rf taiga-back/media /var/www/virtual/taiga/html/
```

4.11 Create `.htaccess` file and put the following content
```
(taiga) [taiga@dysnomia mrr]$ vim ~/html/.htaccess

```

```
RewriteEngine On

# api requests
RewriteCond %{REQUEST_URI} ^/api
RewriteRule ^(.*)$ http://100.64.86.2:63976/$1 [P]
RequestHeader set X-Forwarded-Proto https env=HTTPS

# deliver files, if not a file deliver index.html
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_URI} !^/static/
RewriteCond %{REQUEST_URI} !^/media/
RewriteRule ^ index.html [L]
```

4.12 Enable logs for debugging
```
(taiga) [taiga@dysnomia taiga-back]$ uberspace web log apache_error enable
```

4.13 Use apache as the web backend

```
(taiga) [taiga@dysnomia taiga-back]$ uberspace web backend set --apache ~/html/
```

4.14 Finally good to run the server and test it in the browser

```
python manage.py runserver 100.64.86.2:63976
```

open the url in browser https://taiga.uber.space


#### Notes:

N1. The email configuration for django

```
SMTP_AUTH = True
SMTP_USE_TLS = True
SMTPHOST = 'dysnomia.uberspace.de'
SMTPPORT = '587'


EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
#EMAIL_USE_TLS = True 
#EMAIL_USE_SSL = True # You cannot use both (TLS and SSL) at the same time!
EMAIL_HOST = 'dysnomia.uberspace.de'
EMAIL_PORT = 587
EMAIL_HOST_USER = 'taiga'
EMAIL_HOST_PASSWORD = 'Uj5646()KJd38KKZre'
```

N2. Databases
```
DATABASES = {
     "default": {
         "ENGINE": "django.db.backends.postgresql",
         "NAME": "taiga",
         "HOST": "localhost",
         "PORT": '63921'
     }
 }

```

N3. Sites
```
DATABASES = {
     "default": {
         "ENGINE": "django.db.backends.postgresql",
         "NAME": "taiga",
         "HOST": "localhost",
         "PORT": '63921'
     }
 }

```
N4. Secret key

```
SECRET_KEY = "aw3+t2r(8(0kiaexuko9Li0ahngienai5reifi3baed3RieB5AeN4ceeceu2eehe0la3pei2ceo2okrhg8)gx6i96v5^kv%6cfep9wxfom0%7dy0m9e"

```

References:
1. https://gist.github.com/shuairan/160015f071ef080e841d
2. http://taigaio.github.io/taiga-doc/dist/setup-production.html
3. https://github.com/HepHepHurra/Taiga-on-Uberspace
4. https://github.com/Uberspace/lab/blob/fba013054549ab3f9ebc00e46217f06bc618a41e/source/guide_postgresql.rst
