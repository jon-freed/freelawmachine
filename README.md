Free Law (Virtual) Machine v2.1.0
==================================

This project is designed to provide automation around building a ready-to-run
development environment for the [Free Law Project](https://github.com/freelawproject) using Vagrant boxes.

Hopefully, it will do away with the manual process described
[here](https://github.com/freelawproject/courtlistener/wiki/Installing-CourtListener-on-Ubuntu-Linux)
and make having a dev environment as easy as `vagrant up`.

## Requirements
* [Vagrant 1.8.5](https://www.vagrantup.com)
* [Virtualbox 5.1.4](https://www.virtualbox.org/)
* (Optional) [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
* (Optional) [Packer 0.10.1](https://packer.io/downloads.html)

And a high-speed network connection since the VM will need to pull down
packages! (You are installing Ubuntu...so do this via a cellular data
connection at your own $$ peril!)

## Changes in 2.1.0
* Fixed issues related to Solr core creation/destruction during Django tests
* Added new box version! The new *freelawproject/freelawbox64-desktop* base box
  provides an X11/XFCE4 desktop environment with Chromium installed and ready
  for Selenium tests

## Changes in 2.0.2
* Fix for missing `/reloadCache` Solr endpoint

## New in Version 2.0.0!

Major changes for users of 1.6:
* _A wild Ansible appears!_ Provisioning is now done via Ansible playbooks
  instead of nasty hard to maintain shell scripts.
* The box is set up closer to the original wiki specs, so it uses a Python
  Virtual Environment, but it should be auto-activated for you upon ssh login.
* Similar to v1.6, _you only need the Vagrantfile_ to run the box (assuming you
  have virtualbox and vagrant). It will do the rest.
* *Deprecation Warning!* We're removed support for 32-bit boxes. New standard is
  to use 64-bit to mimic exact same packages as used in Production.

Major changes for users of 1.5 and earlier:
* The `flp` directory is no longer used or needed. If you want to pre-clone a
copy of CourtListener, simply clone it to the directory where the Vagrantfile
resides. _This is not required as the box should do that for you at startup._
* VirtualBox is now told to allocate *2 GB* of memory to the box. This should be
enough for dev use of CourtListener and small enough not to impact most hosts.

# Using the Free Law Machine

There are two ways to use this project:

1. *Running CourtListener* - this is the most common Vagrant task
and is what brings your local Free Law development environment online and ready
for use in development or testing CourtListener.

2. *Building new or custom Vagrant Boxes (Optional)* - if you want to customize
how the box is built or you want to contribute to this project.


# Running CourtListener

1. Make sure you have the base requirements of *Vagrant* and *VirtualBox*
installed and up to date.

2. Either clone this project using git or download a copy of the
[Vagrantfile](Vagrantfile). Copy to a wherever you want to have CourtListener
available.

3. Open a terminal or command line and change to the directory with the
Vagrantfile.

  `cd <directory>`

4. This is where the ✨magic✨ happens! Just run:

  `vagrant up`

  And the base box will be pulled down from it's hosted location, installed
  into VirtualBox, started, and last-mile provisioning steps will take place.
  _This now includes cloning the latest copy of CourtListener for you!_

5. Now to log into the box...it's a simple:

  `vagrant ssh`

For the password, enter 'vagrant'.

6. If you haven't used the machine yet, you'll need to do some basic
CourtListener provisioning steps that currently aren't handled (yet) by our
Vagrant provisioning scripts.

  ``` bash
  cd /var/www/courtlistener
  pip install -U -r requirements.txt
  ./manage.py migrate
  ./manage.py syncdb #to create the admin user
  ```

#  Recommended Next Steps
Here are a couple additional steps to get you started if you're unfamiliar with
CourtListener or your way around a Django app.

## Starting CourtListener
From within the `/var/www/courtlistener` directory, simply use Django's manage
scripts to launch the app. Make sure you pay close attention to adding the IP
address and port number so it plays nice with Vagrant's NAT networking and the
box's network adapters.

```bash
./manage.py runserver 0.0.0.0:8000
```

_For more details on why this is, check out this StackOverflow
[post](http://stackoverflow.com/questions/1621457/about-ip-0-0-0-0-in-django)._

Fire up your browser (on your local machine!) and confirm you've got a local
instance that looks like [courtlistener.com](https://www.courtlistener.com/).

  Navigate to: [http://localhost:8000](http://localhost:8000)


## Scrape Some Court Opinions!

You can easily load some content into your CourtListener instance by scraping
some courts using the [Juriscraper](https://github.com/freelawproject/juriscraper/)
commands built into CourtListener. In either a new SSH session/shell or after
cancelling (`ctrl-c`) the "runserver" command, try:

```bash
./manage.py cl_scrape_opinions \
  --courts juriscraper.opinions.united_states.federal_appellate.ca1 \
  --rate 5
```

CourtListener will spin up a Juriscraper instance for the given court
and load the output into the PostgreSQL instance as well as feed the results to
the Solr instance.

Once complete (after a timeout) or after you manually kill it
with some `ctrl-c`'s, you need to tell Solr to commit changes and make the new
docs in the index go live:

```bash
./manage.py cl_update_index --do-commit --type opinions \
  --solr-url http://127.0.0.1:8983/solr/collection1
```

You should now have some results on the landing page as well as fully searchable
opinions!

You can inspect the Solr index cores directly using your browser:
[http://localhost:8999/solr/#/](http://localhost:8999/solr/#/)

# Building a new Vagrant Box (For Contributors)
Here's how to crank out a box if you've got the Requirements above. Depending
on your network connection, CPU, disk, etc. this could take anywhere from 20
mins to maybe 30 mins. Be patient :-)

All of the tools required to build the box using [Packer](https://packer.io)
are contained in the [packer](./packer) directory of this project. [Ansible](https://github.com/ansible/ansible) is installed into the VM image
itself, so there's no need to have it installed on your local machine.

  0. Grab the latest Free Law Machine source:

    `git clone https://github.com/freelawproject/freelawmachine`

  1. Jump into the packer directory:

    `cd packer`

  2. Build the box! (Yes, it's that simple. Since it's configured headless, it
  may appear nothing is happening for a little while. It's ok. This could take
  about 20-30 minutes!)

    `packer build freelawbox64.json`

  3. Install the Vagrant box on your local machine. The new _.box_ file will
  have a timestamp in the filename, so make sure to add the correct file:

    `vagrant box add freelawbox64-{version}.box --name freelawproject/freelawbox64`

Voila! You now have a new Vagrant box installed locally. You can even share the
_.box_ file with others the old fashioned way, host it at a URL, etc. (Vagrant
supports pulling boxes via URL.)

# Building your own Vagrant box using the Ansible playbooks

If you aren't looking to build a Vagrant base box and instead just want to take
a vanilla Ubuntu 14.04 base image (e.g. something like _ubuntu/trusty64_ or
_/boxcutter/ubuntu1404_), you can install
[Ansible](https://github.com/ansible/ansible) locally and use the same
playbooks used when building from scratch with Packer.

  0. Install the latest Ansible via either `pip install ansible` or other means.

  1. Change into the `ansible` directory where there's already a stubbed-out
  Vagrantfile waiting for you.

  2. The playbooks are executable, so just run: `./freelawmachine.yml` from your
  command line.

  3. Get a drink because you could be waiting about 20 minutes or so :-)


# Vagrant Tips
If you're playing around, here are some things to remember:
* Vagrant installs boxes typically in your home directory under something like
`.vagrant.d`. Make sure you use the same box name when using `vagrant box add`
and it will replace that existing box. (You probably need to add `--force` or
  something to the command, btw.)
* Vagrant keeps instances of vm's in the local directory where your Vagrantfile
is and where you are running `vagrant up`. Do a simple `vagrant destroy` to
wipe it out when you've built a new box version and want to start over.


# Some References
[A packer template for Vagrant from Hashicorp](https://github.com/hashicorp/atlas-packer-vagrant-tutorial.git)
