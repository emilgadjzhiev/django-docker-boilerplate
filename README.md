# Django Docker Boilerplate from PLANEKS

## How to create the project

```shell
$ python3 -m venv venv
$ source venv/bin/activate
$ wget https://github.com/planeks/django-docker-boilerplate/archive/0.9.tar.gz
$ django-admin startproject myproject --template 0.9.tar.gz -e py,html,md,yml -n start
```

After successful project creation you may delete `venv` directory and downloaded
archive.

## How to install Docker and Docker Compose

For Ubuntu 20.04 you can use the next commands for installing Docker:

```shell
sudo apt install apt-transport-https ca-certificates curl software-properties-common
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
$ sudo apt update
$ apt-cache policy docker-ce
$ sudo apt install docker-ce
$ sudo systemctl status docker
$ sudo usermod -aG docker ${USER}
```

The last command is necessary for adding the current user to the `docker` group
for allowing to use the `docker` command without `sudo`.

For installing `docker-compose` use the next commands:

```shell
$ sudo curl -L "https://github.com/docker/compose/releases/download/1.26.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
```

## Running the project on the locale machine

During development, you need to run the project locally. First, of all, copy
the file `local.env` to the file `.env` in the same directory.

```shell
$ cp local.env .env
```

Open the `.env` file in your editor and specify the settings:

```shell
PYTHONENCODING=utf8
DEBUG=1
CONFIGURATION=dev
DJANGO_LOG_LEVEL=INFO
SECRET_KEY="<secret_key>"
SQLITE_DB=/extras/myproject.sqlite3
REDIS_URL=redis://redis:6379/0
STATIC_ROOT=/staticfiles
SESSION_FILE_PATH=/sessions
EMAIL_FILE_PATH=/email
SITE_URL=http://myproject.local:8000
```

We are atrongly recommend to create some local domain in your `/etc/hosts` file
for working with the project:

```
127.0.0.1   myproject.local
```

Also, in this template we are using SQLite for development process. If you will add
the `SQLITE_DB` variable specify the full path to the database file in the container.
In this particular case use the path like `/extras/database_file_name.db`.

We specifying the next volumes in the application container:

- `/extras` -> `data/local_extras`
- `/staticfiles` -> `data/local_staticfiles`
- `/email` -> `data/local_email`
- `/sessions` -> `data/local_sessions`

If you need to add any other directories to the container and define them as
volumes you need to edit `Dockerfile` and `local.yml` file.

Use the next command for building the containers:

```shell
$ docker-compose -f local.yml build
```

For running the project in detached mode use the next command:

```shell
$ docker-compose -f local.yml up -d
```

For running a management command like Django interactive shell, for example, you
can use the next command for running `bash` inside the container:

```shell
$ docker-compose -f local.yml exec django bash
```

## Deploying the project to the server

We are strongly recommend to deploy the project with unpriviledged user instead of `root`.

> The next paragraph describes how to create new unprivileged users to the system. If you use AWS EC2 for example, it is possible that you already have such kind of user in your system by default. It can be named `ubuntu`. If such a user already exists you do not need to create another one.

You can create the user (for example `webprod`) with the next command:

```shell
$ adduser webprod
```

You will be asked for the password for the user. You can use [https://www.random.org/passwords/](https://www.random.org/passwords/)  for generating new passwords.

Add the new user `webprod` to the `sudo` group:

```bash
$ usermod -aG sudo webprod
```

Now the user can run a command with superuser privileges if it is necessary.

Usually, you shouldn't log in to the server with a password. You should use the ssh key. If you don't have one yet you can create it easily on your local computer with the next command:

```bash
$ ssh-keygen -t rsa
```

You can see a content of your public key with the next command:

```bash
$ cat ~/.ssh/id_rsa.pub
```

Now, go to the server and temporarily switch to the new user:

```bash
$ su - webprod
```

Now you will be in your new user's home directory.

Create a new directory called `.ssh` and restrict its permissions with the following commands:

```bash
$ mkdir ~/.ssh
$ chmod 700 ~/.ssh
```

Now open a file in `.ssh` called `authorized_keys` with a text editor. We will use `nano` to edit the file:

```bash
$ nano ~/.ssh/authorized_keys
```

> If your server installation does not contain `nano` then you can use `vi`. Just remember `vi` has different modes for edit text and run commands. Use `i` key to switch to the *insert mode*, insert enough text, and then use `Esc` to switch back to the *command mode*. Press `:` to activate the command line and type `wq` command to save file and exit. If you want to exit without saving the file just use `q!` command.

Now insert your public key (which should be in your clipboard) by pasting it into the editor. Hit `CTRL-x` to exit the file, then `y` to save the changes that you made, then `ENTER` to confirm the file name (in the case if you use `nano` of course).

Now restrict the permissions of the `authorized_keys` file with this command:

```bash
$ chmod 600 ~/.ssh/authorized_keys
```

Type this command once to return to the root user:

```bash
$ exit
```

Now your public key is installed, and you can use SSH keys to log in as your user.

Type `exit` again to logout from `the` server console and try to log in again as `webprod` and test the key based login:

```bash
$ ssh webprod@XXX.XXX.XXX.XXX
```

If you added public key authentication to your user, as described above, your private key will be used as authentication. Otherwise, you will be prompted for your user's password.

Remember, if you need to run a command with root privileges, type `sudo` before it like this:

```bash
$ sudo command_to_run
```

We also recommend to install some necessary software:

```bash
$ sudo apt install -y git wget tmux htop mc nano build-essential
```

And install Docker and and Docker Compose like it has been described before.

Create the new group on the host machine with `gui 1024` . It will be important for allowing to setup correct non-root permissions to the volumes.

```bash
$ sudo addgroup --gid 1024 django
```

And add your user to the group:

```bash
$ sudo usermod -aG django ${USER}
```

Create the directory for projects and clone the source code:

```bash
$ mkdir ~/projects
$ cd ~/projects
$ git clone <git_remote_url>
```

> Use your own correct Git remote directory URL.

Go inside the project directory and do the next for creating some initial
volumes:

```bash
$ source ./init_production_volumes.sh
```

Now you can run the containers:

```bash
$ docker-compose -f production.yml build
$ docker-compose -f production.yml up -d
```

Also, you can setup the Cron jobs for making scheduled backups and cleaning
Docker unnecesary data.

```bash
$ sudo crontab -e
```

Add the next lines

```bash
0 2 * * *       docker system prune -f >> /home/webprod/docker_prune.log 2>&1
0 1 * * *       cd /home/webprod/projects/my_project && /usr/local/bin/docker-compose -f production.yml exec -T postgres backup >> /home/webprod/cronsy_backup.log 2>&1
```
