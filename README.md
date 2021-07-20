# DevOps Primer

This workshop is a primer to beginning system administration and devops. This is
a relatively simple deployment of a hobby project where we will learn to set up
nginx with http 2 compression and tls encryption to serve a dynamic website.
This will also introduce concepts of DevOps such as continuous integration and
continuous delivery using github actions. This workshop is mean't to be a primer
before we do the real enterprise stuff dealing with google cloud, managed
databases, horizontal scaling, load balancers and such.

In this application we are deploying we have a front end and a back end. Both
are seperate applications but both will be deployed on the same server.
Additionally we will have a postgres instance running in the background.

## What is Dev Ops?

Dev ops is kind of like a way of life. It is the idea that we automate a lot of
tedious tasks, so that we can focus on code and automate tasks that would
otherwise be tedious to do on it's own. When we talk about Dev Ops we are really
talking about creating a pipeline, or a continuous flow of code to production.

DevOps is often broken down into two stages, continuous integration and
continuous delivery/deployment.

Continuous Integration typically looks like this.

1. We push our code from our repository to a repository on the internet (e.g.
   github)

2. We trigger our CI instance to run tests on the application to verify that
   it's functions work correctly (called unit testing).

3. We trigger a build, building our application into some runnable form such as
   a binary or a bytecode.

4. We take the output of our build step (called artifacts) and save them,
   deleting everything else. (Really considered a part of the CD phase)

Continuous Delivery and Continuous Deployment are different things and they look
very different.

Continuous Delivery is the more modern approach, in continuos delivery we would
take our build artifacts from the CI phase and build an image of a container or
a service like a digital ocean droplet or a virtual server instance and deploy
it using an orchestration tool (like kubernetes or terraform). We then perform
some tests against our production like
environment (called integration tests, there are also other types of tests),
then we deploy to a high availability environment (something that lots of users
can access 24/7). This is the recommend approach for Dev Ops, it is easy to
scale both vertically (adding more powerful servers) and horizontally (adding
more servers in general). On the other hand it costs a lot of $$$ to maintain
and use this kind of infrastructure. Another use of continuous delivery is just
releasing the binaries as is on things like github releases or publishing to the
app store. This is more for people like games developers who continually build
actual video games, and have no server production environment they need to
manage.

Continuous Deployment on the other hand is taking the artifacts from the CI
phase and automating the deployment directly on the server using the artifacts.
This approach only scales vertically, on the other hand it is great for static
websites using services like github pages, netlify or gitlab pages. It is also a
great option for instances where you would want all your services running on the
same server (if you had a business that used their own servers).

Essentially CD looks like this.

Get Artifacts -> Test against production environment -> Deploy

## Prerequisites

In terms of knowledge it would be helpful but not necessarily required knowledge

- Basic unix/linux file structures and directory structures (not essential)

- Basic unix/linux commands (just look them up if you struggle)

- Vim or Vi text editing (can also use the nano text editor, but vscode is not
  gonna help)

In terms of what software would be required to follow along in case you want to
follow along which is not required, can just listen.

1. A github account connected to your university or instituition (see 2)

2. Would need the student developer pack from github, should take about three
   seconds and a university email to apply for one.

3. Namecheap domain (free with student developer pack and a .me extension) Look
   up instructions on github student pack and google.

4. Some vps (digital ocean droplet recommended) just the account needed setting
   them up will be covered, also doing these things are provider agnostic, i'll
   be using digital ocean but these apply to all platforms, since they all
   provide similar experiences.

5. You NEED some way of getting around, some sort of unix shell. Instructions
   below.

### Windows Unix Shell

In windows there are many options for getting unix shells. Putty is one, but not
recommended for system administration work. The two most recommended options are
to either use a Virtual Machine, but they can be quite tedious to set up, or you
can use wsl. It is recommended that you use wsl.

WSL stands for windows subsytem for linux, and is essentially a linux kernel
embedded in windows allowing windows users access to the linux cli.

In order to get it follow this guide
https://docs.microsoft.com/en-us/windows/wsl/install-win10

I would personally recommend downloading either Debian, Fedora or openSUSE for
WSL.

Debian has the most available packages, but you will only be a few here.
OpenSUSE Leap, is well regarded as one of the best linux distros with good
default tools which would probably be my own personal choice. Then Fedora is
very similar to CentOS which is what we will be using as the deployment
environment, generally it is a good idea to have your development environment be
as close to your deployment environment as possible.

The default packages installed with wsl should be fine.

### MacOS

Make sure ssh is installed and enabled, just the client, not the server.

It is recommended to install vim or a cli text editor.

It is also recommended to install rsync for sysadmin work.

### Linux

See the macos for packages, but everything else should be fine.

## Setting up CI pipelines for testing

### Building a unit testing pipeline in github actions for the rust tasks api

Setting up a CI pipeline we have two goals we need to accomplish

1. We need to Test our application to make sure it functions correctly.

2. We need to build our application for deployment, and test that the build is
   successful.

In order to do that we will create what we call pipelines and tasks in github
actions.

Pipelines are like over arching tasks we need to accomplish, like a testing
pipeline, a build pipeline and a deployment pipeline.

Steps are the steps needed to take in order to achieve the goal set in the
pipeline.

1. Our first step is to fork the tasks api project from the Macquarie DSC's
   github. Navigate to https://github.com/Macquarie-University-DSC/tasks_api_rs
   and click the fork button.

2. Our next step should be to clone the repository into our local computer.
   Navigate to the repository that you have forked. Click on Code, and copy and
   paste the link (or the ssh link if you have set up ssh on your computer), and
   run the command `git clone link_you_copied.git`

3. In your new directory, create the folder `.github/workflows` with
   `mkdir .github && mkdir .github/workflows`

4. Create and open a new file, I will call mine `deploy-actions.yml` in the
   folder `.github/workflows/deploy-actions.yml`

5. We now have to create our first pipeline

Our first pipeline we need to do a couple of things, usually the first step of a
CI pipeline is running unit tests, this just tests that each individual function
of our web app works correctly. We need to test that functions such as creating
new id's and then deleting them from a database works as expected, to do this we
need to have a database instance running in our pipeline.

```yaml
name: Deploy Actions
on:
  push:
    branches:
      - master
jobs:
  unit-tests:
    runs-on: ubuntu-20.04
    services:
      postgres:
        image: postgres:13.3-alpine
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: tasks_db_test
        ports:
          - 5432:5432
        options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
    env:
      DATABASE_URL: postgres://postgres:postgres@localhost/tasks_db_test
    steps:
      - uses: actions/checkout@v2
      - uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
          components: clippy
      - uses: actions-rs/install@v0.1
        with:
          crate: sqlx-cli
          version: latest
          use-tool-cache: true
      - run: cargo sqlx migrate run
      - run: cargo clippy
      - run: cargo test
```

Let us break this down.

The name is what we want to call our pipeline, for large projects we could have
multiple pipelines doing different things for most people they would only have
one.

Jobs is the steps in the pipeline that we need to take, so far we only will have
one step, running unit tests.

remember our steps

checkout from source control -> run unit tests -> build application

So first we create a job, called unit-tests, which we run our unit tests in, in
this job we run on a linux environment, github actions only supports running on
ubuntu. So we pick `ubuntu-20.04`, this is the latest ubuntu as of writing this
supported by github actions. We also freeze all our testing platform for
consistancy in the future.

We need to have a test database setup and configured, first we need to configure
a docker image that will run in the background.

Our docker image uses alpine, a lightweight linux distribution for containers,
and is hard locked to use 13.3 for consistancy. We set a few environment
variables on our postgres image, setting the username and password to postgres
and setting the database name to be tasks_api_test. We also need to forward the
`5432` port so that our test can access the database, and the last option checks
if the database is ready before continuing on to the next step.

We set the postgres port of our application in the format required by sqlx. This
is pretty simple and uses the configuration specified when we set up our
database.

Now we need to define the steps in order to actually test the application. If
you notice the steps, some are `run` commands and some are `uses` commands.
Github has a collection of github actions for doing certain steps. Like we need
to copy our project into our testing environment, github already has a pipeline
for this called `actions/checkout@v2`.

Our second uses is to install the rust toolchain, it contains everything
typically needed to build and test a rust application. We install the latest
stable version, and we also want to install `clippy`. Clippy is a linting tool,
while building our application will test that the code compiles correctly,
clippy will go a step further and warn us about any performance problems. Clippy
also warns us about any style errors and syntax issues such as incorrect naming
conventions and breaking style rules. After that we install the sqlx-cli for
creating our database.

From our three run commands the first command is to create a database for
testing purposes. The second command is to run linting on our application. Our
last command is to run the actual unit tests.

The biggest downside to using rust is this testing process due to compile times
and the very low resources available on the ci platform will take about 10
minutes :(.

### Creating a build pipeline in github actions for rust api

Our next step in building our ci pipeline is to create a build step. Most would
consider this part of the CI phase since we are not actually deploying anything
yet. We are just testing if our application builds successfully.

In this step due to the limitations of centos 7 which uses a very very outdated
version of the c standard libraries, we need to completely statically compile
our application. By default rust will only statically compile the rust
components of our application and will not statically compile the c standard
library which rust applications use for things like memory allocation. So we
need to install musl which is a library that is a lightweight copy of the gcc
c standard library that can also be statically linked.

#### Writing the build pipeline

Our actual build pipeline is relatively simple with a few gotchas. We need to
checkout our source code first of course for obvious reasons.

Next we need to install the dependancies for musl on ubuntu `musl` and
`musl-tools`.

We also have to install the musl target rather then just the regular target.

We build our application in release for optimisation and removing debug code.

Our last step is to upload the result as a build artifact for when we deploy our
application.


Our actual build pipeline would be pretty simple. Our first step is to add encrypted secrets to our github actions CI.

Go to your repository where you have your CI setup, go to Settings -> Secrets and click New repository secret.

Remember the command to generate a secret. Run that again and copy, call this secret `SECRET_KEY_BASE` and it's contents
should be the random string you have generated.

Now remember your database URL, create a new repository secret and it's name should be `DATABASE_URL`, and add your
database url to the contents.

Let us now update our actions workflow to look like this.

```yaml
name: Deploy Actions
on: [push]
jobs:
  unit-tests:
    runs-on: ubuntu-20.04
    services:
      postgres:
        image: postgres:13.3-alpine
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: tasks_db_test
        ports:
          - 5432:5432
        options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
    env:
      DATABASE_URL: postgres://postgres:postgres@localhost/tasks_db_test
    steps:
      - uses: actions/checkout@v2
      - uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
          components: clippy
      - uses: actions-rs/install@v0.1
        with:
          crate: sqlx-cli
          version: latest
          use-tool-cache: true
      - run: cargo sqlx migrate run
      - run: cargo clippy
      - run: cargo test
  build:
    needs: unit-tests
    runs-on: ubuntu-20.04
    steps:
      - uses: actions/checkout@v2
      - run: sudo apt-get install musl musl-tools
      - uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
          target: x86_64-unknown-linux-musl
      - run: cargo build --release --target x86_64-unknown-linux-musl
      - uses: actions/upload-artifact@v2
        with:
          name: _build
          path: target/x86_64-unknown-linux-musl/release/tasks_api_rs
```

So this is what our build step looks like. As a rust project and compiling in
release mode, it will take so fricken long to compile and test so RIP.

Now we have pretty much finished our CI pipeline, pretty easy stuff I reckon!

### Building unit testing pipeline in github actions for elm app

Testing front end functional web applications is so much easier. They are not
dependant on a storage device like a database, and have no state. As such the
config would be a lot smaller.

As such our testing pipeline is going to be a lot smaller then our rust testing
pipeline.

```yaml
name: Deploy Actions
on:
  push:
    branches:
      - deploy
jobs:
  unit-tests:
    runs-on: ubuntu-20.04
    steps:
      - uses: actions/checkout@v2
      - uses: borales/actions-yarn@v2.3.0
        with:
          cmd: install
      - uses: borales/actions-yarn@v2.3.0
        with:
          cmd: test
```

So essentially all we are doing is grabbing our source code from github, then we
install all our dependancies using yarn. After that we run the test script which
runs elm-test on our application.

### Building a build pipeline in github actions for elm app

Just like how the steps to test our frontend were a simplified version of the
steps to test our server. The steps to build our frontend are also a simplified
version of the steps to build our backend.

```yaml
name: Deploy Actions
on:
  push:
    branches:
      - deploy
jobs:
  unit-tests:
    runs-on: ubuntu-20.04
    steps:
      - uses: actions/checkout@v2
      - uses: borales/actions-yarn@v2.3.0
        with:
          cmd: install
      - uses: borales/actions-yarn@v2.3.0
        with:
          cmd: test
  build:
    needs: unit-tests
    runs-on: ubuntu-20.04
    steps:
      - uses: actions/checkout@v2
      - uses: borales/actions-yarn@v2.3.0
        with:
          cmd: install
      - uses: borales/actions-yarn@v2.3.0
        with:
          cmd: dist
      - uses: actions/upload-artifact@v2
        with:
          name: _site
          path: dist/
```

Coming from testing, and repeating the process, our application just replaces
the test command with dist, which is a script that runs `parcel build`. Parcel
will automatically compile and minify all our assets for production use.

After that we upload the build artifacts in the `dist` folder.

Easy stuff.

## Initial server setup for deployment

### A quick note on choosing linux distributions

For production servers we generally think about performance, ecosystem, support
and stability. Generally speaking we don't really care about performance for web
applications like these where performance is not a demand. Looking at the
ecosystem, linux is linux and bsd is bsd, they have mostly the same tools and
mostly the same ecosystem. That being said CentOS is based on Red Hat Enterprise
Linux, so it has tools and is tuned towards enterprise server management. CentOS
is a free and open source fork of a proprietary linux distribution, sometimes
people would rather a completely community maintained distribution such as
Debian, or FreeBSD. On the other hand support from big companies like Red Hat
and SUSE is a big deal. Support can mean providing essential security patches
when they or needed, or services specific to that distribution. Another key part
of support is the support period of a distribution, CentOS shines in this area
with a support period of 10 years ending in 2025 for CentOS 7. This ties into
security and stability, in linux distributions for production environments we
prefer fixed release distributions, as they do not have as many dependancy
changes so we can ensure our server runs consistantly.

CentOS is run by RedHat and has had some major controversy recently. CentOS 7 is
the last long term support CentOS distribution, previously CentOS had been a
downstream fork of RHEL, it was a copy of RHEL distributions, but recently that
has changed where CentOS has become a testbed for RHEL distros. As such two
Linux Distros have come along as community maintained alternatives to fill in
the void of CentOS, Rocky Linux and Alma Linux. I recommend checking these
distributions out since they are really cool!

### Creating a CentOS 7 Droplet

There are many options for hosting websites, digital ocean and vultr from my
experience would be by far the easiest to use. That isn't to say that the
other's aren't good, but they are generally for larger scale applications.

First log into your digital account, then

1. Click the get started with a droplet button

2. Select CentOS 7.x as your distro of choice.

3. Select a relevant spec for your droplet, anything will do it is a matter of
   cost and personal preference.

4. Choose whatever region you would like, I use singapore since it is the
   closest to Sydney.

5. Choose the password option instead of the ssh one, don't select User Data.
   Some users of digital ocean might like to setup these options up front, but
   it is my personal preference not to.

6. Choose a hostname for your droplet or leave it as is.

7. Make sure to click the backups option.

8. Now wait for the droplet to be created.

### Setting up the URL for our website

So for getting our domain name, I would use the free namecheap domain provided
through the github student developer pack.

Why?

1. It's free

2. It has whois guard protection, if someone does a whois on your domain, they
   will see a bunch of random text rather then your actual personal information.

3. All domain name providers will have essentially the same steps.

Assuming you have already selected a namecheap domain name, we need to setup
digital ocean dns servers, so that we can use digital ocean to redirect our
domains to our web server.

1. Log into namecheap.com and navigate to the Dashboard.

2. Navigate to your domain, and select manage.

3. Under name servers, change to custom dns, and set the nameservers to the
   following

```
ns1.digitalocean.com
ns2.digitalocean.com
ns3.digitalocean.com
```

4. Log back into digital ocean and on the `...` next to your droplets name,
   select add a domain

5. Add these records

```
A Record:
hostname: @
will direct to: your droplet

CNAME Record:
hostname: www
is an alias of: @

A Record:
hostname: api
will direct to: your droplet
```

### Setting up SSH

Setting up SSH is pretty easy

1. Run the command `ssh-keygen` and simply press enter through all the options.
   Note that if you would like to set a password that is fine.

2. Next add ssh to your github account, click your profile icon in the top
   right, select `settings`, then under SSH and GPG keys click New SSh key on
   the top right. Copy the your key with the command `cat .ssh/id_rsa.pub` this
   will print your PUBLIC key to the terminal, as `cat` prints files to the
   terminal, and `.ssh/id_rsa.pub` is the location of where your key is stored.
   ssh keys have a private and a public component. The private is the key and
   only you can access it, the public key is like providing the lock that they
   key fits so never give your private key away.

3. now in your server instance, find what the servers public ip is (remember to
   provide instructions for this) and run the command
   `ssh-copy-id root@yourdomain` where the server ip is the public ip address to
   your server. (Note: sometimes it takes a while for domains to get registered
   when switching nameservers, in that case use the ip)

We done for now.

### Users and Permissions

In linux we have users and groups

Linux contains a simple permissions model, each service has permission information, they can be read, written, or
executed (like opening up an app in windows). You can view permission with the `ls -a` command, it appears with
something of the form of `drwxrw-r-- user group`, the first letter `d` tells us whether it is a directory or not. The
next three characters are in the form of `rwx` where `r` is the read permission, `w` is the write permission, `x` is the
execute permission, and `-` says they do not have the given permission. A file often belongs to a user and a group, the
permissions tells us what permissions the user who owns the file has, what permissions the group who owns the file has
and what permissions everyone else (others) has.

Example:

`rw-` - Read and Write permissions but no execute.

`r-x` - Read and execute but no write permission.

Note that `rwx` is repeated three times, the first time is the read, write and execute permissions for a user, the
second is for a group, and the third is for the entire system commonly called others.

In linux we want to restrict access as much as possible to prevent things from having access to other things they
shouldn't.

In linux a user is like a user of an account like in windows or mac, and a group is just a group of users. When we add
permissions to a specific user, only that user can access that resource, but if we need some users to access a resource
but others not be able to access a given resource, then we can create a group. 

Note on `gpasswd` vs `usermod`

There are two ways to add a user to a group, `usermod -aG somegroup username` and `gpasswd -a username somegroup`

The difference is subtle, but `usermod` modifies the users configuration with options such as changing there default
terminal shell, or adding and removing groups.

`usermod` has two options that we will look at `-G` which gives a user certain groups, and `-a`, which adds the
pre-existing groups that a user had to the user. Now the problem is if you accidentally forget the `-a` flag, you end up
removing the groups a user alread had, and so this can be considered more risky, in practice this rarely happens. On the
otherhand `gpasswd` can either add or remove groups so it is therefore a safer option.

1. run the command `adduser exampleuser` where the user is your username in my case emendoza.

2. set a passwd for your user with the command `passwd exampleuser`

3. run the command `gpasswd -a exampleuser wheel` this adds our user to the wheel group. In Linux, the wheel group is
   like the admin role in windows.

4. Logout of ssh with Ctrl+D or type `exit` into the cli.

5. Now repeat the command `ssh-copy-id exampleuser@yourdomain` notice we are now using the example user instead of root.

### Install required software

1. Run command `sudo yum upgrade` to make sure we are downloading the latest versions.

2. Run command `sudo yum install vim-enhanced`

3. Run command `sudo yum install epel-release` to install the red hat extra packages.

### Note on systemctl and services

All operating systems have software that runs in the background, they do things like make sure that the time is
synchronised with the rest of the world, they provide resources for apps that run in the foreground. These are commonly
called Daemon services.

In windows it is kind of difficult from my experience (not a windows user tho) to create and manage services. If we want
to make an app that runs when we start up our computers and runs in the background it can be quite tedious. Linux on the
other hand has a tool that allows us to do this, it is called systemd and can be managed using systemd service files
usually with the extensions of `.socket` and `.service` and managed with `systemctl`. Read more on the
[archwiki page](https://wiki.archlinux.org/index.php/systemd) 

Systemctl has the commands

- `systemctl start servicename` to start a service called servicename
- `systemctl stop servicename` to stop a service called servicename
- `systemctl restart servicename` to stop then start a service called servicename
- `systemctl status servicename` to see if the service activation actually worked
- `systemctl enable servicename` to make sure that the service starts on system start up e.g. when
  you turn on your computer or when you restart your computer.

You can view the systemd logs with the `journalctl` command.

`.socket` files often are services that start when they are needed, like a printing service only starts when you need to
print something, and `.service` files are for services that run all the time.

You can also specify services for specific users only but read the archwiki for that.

### A note on the linux filestructure

The linux filesystem structure is a standard created by the linux foundation and specified under the Filesystem
Hierarchy Standard, more information can be found on
[the wikipedia page](https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard)

This isn't too important here except when we get to setting up nginx, and is a bit of a point of contention among
system administrators. To understand why there are some important directories which we have to explain.

The root `/` is where everything goes into, equivalent of the `C:` drive in windows.

The `/etc` directory is where applications put there global configurations, that when they start all configuration
options and files are mean't to be put here.

The `/usr` directory specifies read only files used by apps, a common example would be icons, music or stuff in a game,
they are read only because they never need to be modified.

The `/var` directory is kind of like the read and write counter part of the `/usr` directory, things that do need to be
modified but aren't configuration files often go here.

The `/srv` directory is what causes the controversy, according to the linux standard, files which are mean't to be used
for servers such as directories which are used as remote file storage like google drive kind of things go here,
and static website files.

In nginx it has become standard to put your website in the `/var/www` directory, this to me personally isn't the right
place to put them, some system admins share this opinion while others feel that it is better to follow the standards
of nginx for consistancy. Nearly all guides on the internet will use `/var/www` while we will be using `/srv/www`
instead.

### Change ssh settings

We will use vim for this, if you are unfamiliar use replace `vim` with `nano`, but remember, as described in the Unix
and Linux System Administration Handbook System administrators will judge you so do it descretely if you plan to deploy
infront of other system admins.

1. enter the command `sudo vim /etc/ssh/sshd_config`

2. change the line `PermitRootLogin yes` to `PermitRootLogin no`

3. change the line `PasswordAuthentication yes` to `PasswordAuthentication no`

4. quit vim with `:wq`

5. reload ssh daemon with `sudo systemctl reload sshd`

6. Test it works before you exit by entering a new terminal and typing `ssh exampleuser@yourdomain`

### Firewall

A server works by exposing all ports available to the outside world. As system administrators one of our primary jobs is
to restrict the access of the outside world to be able to only access what they need. So we can directly access our
server, but nobody else should be able to, similarly when we serve our website, we want people only to be able to access
what they need to see the website, we don't want them to be able to access anything else. A firewall makes it so that
people can only access ports that we specify, with all other ports still being accessible internally.

#### About firewalld

For this workshop we will use firewalld, it works by having default zones, a zone is an area that the network runs in,
for example, you might have a zone for local computers in an office, and a zone for computers outside in the public, or
a zone for just the computer specifically. Each zone might have certain services enabled such as you might have http
enabled publically for everyone, but only have ssh enabled for users within the local network.

More information about firewalld can be found by googling archwiki firewalld, and you can see what commands are
available by reading the manual with `man firewall-cmd`.

1. Run command `sudo yum install firewalld`

2. start firewall service with `sudo systemctl start firewalld`

3. add ssh to the firewall permissions list with `sudo firewall-cmd --permanent --add-service=ssh`

4. type `sudo firewall-cmd --relaod` to enable changes

5. similar to before open up a new terminal and see if you can still access the server.

6. if it all works enable changes with `sudo systemctl enable firewalld`

### Timezones

It is worthwhile to change the timezone of the server to the local timezone you are in especially dealing with databases
and such. As a side note I often change the local timezone of my databases and api's to UTC and let region specific time
information be handled on the client side.

1. Find which timezone you want by running the command `sudo timedatectl list-timezones`

2. Navigate with the j and the k keys `j` for up and `k` for down. You can also search with the `/` command. for example
   `/Australia` for all regions in australia. Quick tip, try running `sudo timedatectl list-timezones | grep Au` to
   search for timezones in Australia.

3. Next set your region as default with `sudo timedatectl set-timezone region/timezone`, for example,
   `sudo timedatectl set-timezone Australia/Sydney`

4. Confirm with `sudo timedatectl`

### A quick note on SELinux

We will have our first experiences dealing with the pain in the ass that is selinux. SELinux stands for Security
Enhanced Linux and is like an extra more intense linux permissions layer. Instead of files having just read, write and
execute permissions they also have policies about what they are allowed to do on the system and what they are used for.
SELinux on the scale we are using here isn't too bad, and since it is so common for linux files and folders to be used
for what they are, most of CentOS already has default SELinux policies that we need to enable or restore manually. As a
website scales, SELinux becomes harder and harder. Learning SELinux is kind of a thing on it's own and so I wouldn't
stress about it.

SELinux has three modes

- Enabled: if a selinux check fails the service cannot run
- Permissive: if a selinux check fails an error is logged but service still runs
- Disabled: selinux policies aren't checked at all.

Generally it is a good idea to turn selinux to permissive, while deploying a server and change it to enabled later.
[Changing Selinux state guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/using_selinux/changing-selinux-states-and-modes_using-selinux)

### Postgresql

We need to setup a database to store all our data. This is relatively simple.

1. Postgres already exists in centos, but centos 7 ships with an out of date and unsupported version of postgres. So we
   need to exclude postgres from the centos default repo. Edit the repo config with
   `sudo vim /etc/yum.repos.d/CentOS-Base.repo` and change the default config to the following:

```
[base]
name=CentOS-$releasever - Base
mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os&infra=$infra
#baseurl=http://mirror.centos.org/centos/$releasever/os/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

exclude=postgresql*

#released updates
[updates]
name=CentOS-$releasever - Updates
mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=updates&infra=$infra
#baseurl=http://mirror.centos.org/centos/$releasever/updates/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

exclude=postgresql*
```

2. Install the postgres repo with
   `sudo yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm`

3. Install postgres with the command `sudo yum install postgresql13-server`, there are a lot of y's that need typing so
   optionally tell centos to click yes for everything with `sudo yum install -y postgresql13-server`

4. Initialize the database with `sudo /usr/pgsql-13/bin/postgresql-13-setup initdb`

5. Start postgres with `sudo systemctl start postgresql-13`

6. Check postgres is working with `sudo systemctl status postgresql-13`

7. Enable postgres on restart with `sudo systemctl enable postgresql-13`

8. Now we need to switch to the postgres user in order to setup our production database with `sudo -iu postgres`

9. Remember our database url? Now we need to remember what we called our user, and type `createuser --interactive`
   1. We want to call our user what we called our user in our database url.

   2. We do not want our user to be a super user (type n)

   3. We do not want our user to be able to create new databases (type n)

   4. We do not want our user to be able to create new roles (type n)

10. We now want to create a new database with `createdb tasks_db` where tasks_db is what you called your database in the
    database url.

11. Log into the postgres interactive shell, with `psql`. Note that is is a sql terminal so commands need to be
    terminated with a semicolon `;`.

12. Now we need to set the password we set in our database url to our user, do this with the command
    `ALTER ROLE youruser WITH PASSWORD 'yourpassword';`

13. Now we need to grant our user permissions to manage the database with
    `GRANT ALL PRIVILEGES ON DATABASE tasks_db TO emendoza;`

14. Exit with Ctrl+D once to exit psql and again to return to your users shell

15. Manage database with `psql -d tasks_db`

16. Run the command `SET TIME ZONE 'UTC';` and press Ctrl+D to finish database setup.

We are now done setting up postgres.

### Nginx

Nginx is kind of like the server part of our server, we use it to manage routes and http access to our server. Most
backend web frameworks like flask, django, actix, and others don't require it since they can serve the html files 
themselves. On the other hand I often use it anyway because it can be used to easily add advanced features like
compression, https and http2 to our web pages. In our example we will have three domains that nginx manages
www.howgood.me, howgood.me and api.howgood.me. All three will use gzip compression and http2, and api will redirect to
our api endpoint, where www.howgood.me will be our site for users to access, and howgood.me will redirect to
www.howgood.me.

#### Installing nginx

Nginx is in the extras repo for centos which we already installed before.

1. Simply type `sudo yum install nginx`

2. Start nginx with `sudo systemctl start nginx`

3. Check if nginx is working with `sudo systemctl status nginx`

4. If everything works enable with `sudo systemctl enable nginx`

5. Now add nginx to firewall with `sudo firewall-cmd --add-service=http`

6. Also add https for later `sudo firewall-cmd --add-service=https`

7. Save changes with `sudo firewall-cmd --runtime-to-permanent`

#### Setting up blocks in nginx

In nginx you can set up 'blocks' which point to specific domains, that means you can have multiple different domains
that point to the same server!

We will only be using a single block so it's kind of pointless but it's handy for using later if for example you wanted
to deploy a static website and an api on the same server.

1. Make a server staging configurations directory with `sudo mkdir /etc/nginx/sites-available` this is where we write
   our server configurations.

2. Make a server deployment configurations directory with `sudo mkdir /etc/nginx/sites-enabled` this is where our
   finished configurations go.

3. Now we need to modify the `/etc/nginx/nginx.conf` file, I will use the command `sudo vim /etc/nginx/nginx.conf`
replace vim with nano if you aren't comfortable with vim.

Go to where is says `http {` and scroll to below the matching `}` bracket and add the following lines after the bracket.

```
include /etc/nginx/sites-enabled/*.conf;
server_names_hash_bucket_size 64;
```

4. last step is to restart nginx to enable changes with `sudo systemctl restart nginx`

Navigate to your website e.g. http://www.somedomain.me in your browser and see if the installation worked correctly.

### Setting up remote deployment

When we remote deploy something, that refers to deploying 
1. Create a new ansible user with `sudo useradd ansible` and `sudo gpasswd -a ansible wheel`

2. Create a ssh key directory with `sudo mkdir /home/ansible/.ssh`

4. Open a new terminal and create a directory called `priv_keys` here you will store ssh keys temporarily

5. Run `ssh-keygen -t rsa -f ./priv_keys/id_rsa`. Just like a normal ssh keygen.

6. Run the command `cat ./priv_keys/id_rsa.pub` and copy the output.

7. Back in the server type the command `sudo vim home/ansible/.ssh/authorized_keys` and copy and paste the key using
   insert mode.

8. Also copy `cat ~/.ssh/id_rsa.pub` to this file.

9. Now we have to set the correct permissions, with `sudo chown -R ansible:ansible /home/ansible/.ssh` which sets the
   owner of the .ssh folder in ansibles home directory to be ansible. Run the command
   `sudo chmod -R 600 /home/ansible/.ssh` to set the folder to only have read write permissions by the ansible user.

Congratulations we have finished our initial server setup.

## Building and Testing our ansible configuration and final server setup

Our first step to continuous deployment is using ansible to setup our server for continuous deployment. We will be using
ansible to automate the installation of our server.

### What the heck even is ansible and vagrant?

Ansible is what we call a configuration management system, it allows us to quickly write scripts similar to devops
pipelines in github actions that quickly setup and install software. Setting up continuous deployment is not a task of
just copying and pasting files to a remote server, it involves changing user permissions and installing and updating
software within the system. We could of course write shell scripts to automate this, and this is the norm actually, but
as your project grows, this quickly becomes quite hard to manage a script file. Configuration management is easier to
grow and improve upon. There are two problems involved with configuration management though. One is where you can easily
mess up your system, and the other is where having a single source that sets up a system is a massive security risk. Not
dealing with security today, but we need to atleast demonstrate creating a configuration in a testing environment that
is similar to our production environment but not exactly the same. This is where vagrant comes in.

Vagrant is a frontend for virtual machines. It allows us to quickly set up virtual machines for testing things like
ansible configurations. It is really easy to use, and you can get a centos/7 instance up and running in three commands.
Although I could not be bothered to test our ansible configurations so I will not be using vagrant :(.

### Creating ansible configurations for our API

Ansible configurations are called plays and the ansible scripts we write are called playbooks. Our api needs two
playbooks. One playbook is to setup our database with our application, the other playbook is to move our binaries to the
local folder, setup proper permissions and setup our server to run our program as a systemd daemon.

1. In the tasks_api project create a folder to put your ansible stuff in. `mkdir ansible`.

2. First we need to define what our servers are. We define these in ansible 'hosts' or 'inventory' files. These have
   names of our servers that we need to deploy. Create a hosts file in the ansible directory with these settings.
   `howgood.me ansible_user=ansible`

3. Now create a playbook called `migrate.yml`. `touch ansible/migrate.yml` and add the following configuration:

## Setting up a Continuous Deployment Pipeline

