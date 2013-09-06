zero-to-7zip
============

The tale of Docker, Salt Stack, Windows Server 2012, and a whole lotta WIN.

Install the AWS CLI
-------------------

While not strictly necessary, it can be very useful to have around and will
make copy/paste instructions much easier :o]

Detailed instructions can be found at https://github.com/aws/aws-cli

Run the python pip installer:

    pip install awscli

Edit your bashrc and set a path for the default configuration:

    mkdir ~/.aws
    cat > ~/.aws/config.ini <<END
    [default]
    aws_access_key_id=<Your AWS Access ID>
    aws_secret_access_key=<Your AWS Access Secret>
    region=us-east-1
    END
    chmod -R go-rwx ~/.aws
    echo 'export AWS_CONFIG_FILE="~/.aws/config.ini"' >> ~/.bashrc
    . ~/.bashrc
    complete -C aws_completer aws

Test out your setup by listing out your existing AWS instances:

    aws ec2 describe-instances

Kick Off the Windows 2012 Instance
----------------------------------

Windows instances can take up to 30 minutes to 'spin up'. So we'll kick
this off now and come back to it later.

    aws ec2 run-instances --image-id ami-60c0bc09 --key-name='<Your Key>' --security-groups '<Your Security Group>' --instance-type t1.micro

CentOS Salt Master
------------------

Kick off an Ubuntu 13.04 instance with the Docker configuration:

    aws ec2 run-instances --image-id ami-7d317314 --key-name='<Your Key>' --security-groups '<Your Security Group>' --instance-type t1.micro --user-data '#include http://get.docker.io/'

SSH into your newly minted Docker instance:

    ssh -i '<Your AWS SSH Key>' ubuntu@<Docker FQDN>

Become root:

    sudo -i

Make sure you're up to date:

    apt-get update
    apt-get upgrade

Pull down the Salt Master container:

    docker pull calebcase/salt-master-centos

Do a dry run; This is going to fail complaining about fork/exec not
permitted. ctrl-d to exit the run. Some missing files have been created
and that error won't happen again.

    docker run -i -t calebcase/salt-master-centos /bin/bash

Ok run it for real:

    docker run -i -t -p 4505:4505 -p 4506:4506 calebcase/salt-master-centos /bin/bash

Now you're in a CentOS container with Salt Master installed. I've taken
the liberty to also pull down the Windows package repo. See
https://github.com/saltstack/salt-bootstrap and
https://github.com/saltstack/salt-winrepo

At this point we can bring up the salt master and local minion:

    service salt-master restart
    service salt-minion restart

A quick note about key management. This image already has keys setup for the
master and minion. This is totally insecure because I have just shared the
master's private key with you... If you want a secure master you'll need to
regenerate the keys:

    salt-run manage.key_regen
    service salt-master restart

Watch for the minion to check in with the master with the new key:

    watch 'salt-key -L'

When you see the unaccepted key for docker.salt.master exit the watch (ctrl-c)
and accept the key:

    salt-key -ya docker.salt.master

Check that salt can contact the local minion:

    salt '*' test.ping

Expect to have output like:

    docker.salt.master:
        True

Windows 2012 Salt Minion
------------------------

By now your Windows instance might be available. Get yourself an RDP viewer (I
use CoRD on OSX) if you don't already have one.

Extract your login information from the AWS console.

Login via RDP and open a browser. Navigate to
http://docs.saltstack.com/topics/installation/windows.html and find the most
recent installer for AMD64. Download that and run it!

For the master put in the public FQDN of the docker salt master. Change the
minion ID from 'hostname' to windows.salt.minion (this will make it easy to
refer to later).

If things get bungled up check C:\salt for the executables, logs and
configuration. Sometimes it is necessary to restart the salt minion service.

Back on the salt master:

    salt-key -L

You'll see an unaccepted key from this windows minion:

    Accepted Keys:
    docker.salt.master
    Unaccepted Keys:
    windows.salt.minion
    Rejected Keys:

Accept the key:

    salt-key -ya windows.salt.minion

Test your connection to the new minion:

    salt '*' test.ping

You'll see the following (there may be a delay so try a couple of
times):

    docker.salt.master:
        True
    windows.salt.minion:
        True

Now we're ready to install some packages!

Install 7zip
------------

From the salt master, update the windows package repos:

    salt-run winrepo.update_git_repos
    salt-run winrepo.genrepo
    salt '*' pkg.refresh_db

Install 7zip:

    salt windows.salt.minion pkg.install 7zip

From the windows minion, check for 7zip.

That's it. Seriously.

Check /srv/salt/win/repo for additional Windows installers and for examples of
how to create your own.

Extra Credit
------------

You might be wondering what happens if you try to install on both Windows and
Linux? If the package name is the same in the Windows repo and the Linux
package manager (yum, dpkg, etc):

    salt '*' pkg.install putty

Now you have putty installed on both the Windows minion and the salt master.

Don't want it anymore?

    salt '*' pkg.remove putty

Of course, if they don't match up nothing will happen on the machines where
that package doesn't exist. For example try '7zip'.
