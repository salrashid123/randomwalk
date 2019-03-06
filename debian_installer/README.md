
# Creating a simple Debian installer

I usually cover topics related to Google Cloud but from time to time some random tasks that took me a fair amount to figure out.   In the spirt of the latter, i found myself creating my very first packaged installer for linux.  It wasn't done for any particular customer-related critical issue but because a collegue of mine asked about how to easily install `OpenCensus` agent on a debian system and manage its service lifecycle (and the prospect of fine beer :).  That started me down the 8+ hour path in writing a package installer and distribution site from scratch and i thought i'd document a simple how-to as sort of a personal log.

Ofcourse its nothing remarkable or novel or specifici to GCP but here it is.  This article:

* creates a debian `.deb` installer for [OpenCensus Agent](https://opencensus.io/agent)
* creates a package distribution site (i.,e apt-repository; just like the professionals!)

The references below cover both steps i ran on my laptop to generate the `.deb` and on Google Cloud Platform VMs to host the deb file and one other VM to consume (install it via apt)

>> Note:  big disclaimer:  this is the first installer i've written...If you plan to use this for any real applicaiton, please read the official documentation carefully...this tutorial will just help you get started.

## References

- [Debian-Installer: Building the installer yourself](https://wiki.debian.org/DebianInstaller/Build)
- [Required files under the debian directory](https://www.debian.org/doc/manuals/maint-guide/dreq.en.html)
- [Creating your own Signed APT Repository and Debian Packages](http://blog.jonliv.es/blog/2011/04/26/creating-your-own-signed-apt-repository-and-debian-packages/)
- [Debian Packaging Tutorial](https://www.debian.org/doc/manuals/packaging-tutorial/packaging-tutorial.en.pdf)


## Creating a .deb file

So what will out installer do:

* package the opencensus agent.  The agent is already provided as a [platform binary](https://github.com/census-instrumentation/opencensus-service/releases) so this article does not compile from source
* create user/group that the agent will run as `oc-agent/oc-agent`.
* install the agent and cofig file under `/opt/opencensus-agent/*`
* install and configure `systemd` such that you can start/stop the agent easily.

Lets get started.  As mentioned in the introduction, i'm running all the steps using docker. Everything: the build environment, the repository server _and_ the client system that will install the `.deb` file.

1) Setup some libraries we will need later

```bash
apt-get update
apt-get install debhelper dh-systemd dh-make reprepro gnupg2 vim wget nginx lsb-release -y
```

2) Generate a GPG key or use the one provided here (note, the password is `opencensus0!`)

```bash
gpg --import stage/certs/oc_private.key 

  gpg: /root/.gnupg/trustdb.gpg: trustdb created
  gpg: key 0C74009E141D7911: public key "OpenCensus (https://opencensus.io) <census-developers@googlegroups.com>" imported
  gpg: key 0C74009E141D7911: secret key imported
  gpg: Total number processed: 1
  gpg:               imported: 1
  gpg:       secret keys read: 1
  gpg:   secret keys imported: 1

gpg --import stage/certs/oc_public.key  
  gpg: key 0C74009E141D7911: "OpenCensus (https://opencensus.io) <census-developers@googlegroups.com>" not changed
  gpg: Total number processed: 1
  gpg:              unchanged: 1
```

(or create your own key `gpg --full-generate-key`)
- pick sign only (option 4);
- Username: `Opencensus`
- email: `census-developers@googlegroups.com`
- pick any password

Now note down the `keyID`.  In the example below, its `141D7911`

```
gpg --keyid-format SHORT --list-keys
  pub   rsa4096/141D7911 2019-03-01 [SC]
        9A70363996C3EE279ABBAF2E0C74009E141D7911
  uid         [ unknown] OpenCensus (https://opencensus.io) <census-developers@googlegroups.com>
```

3) Create debian installer root folder

Create the folder to house our configuration files.  I'm creating `opencensusagent` version `0.1.2` so i named the directory
exactly in that format with a hyphen (the folder structure/name convention is important).

```bash
mkdir opencensusagent-0.1.2/
cd opencensusagent-0.1.2/
export USER=`whoami`
```

4) Use `dh_make` to to prefill the installation files

[dh_make](https://manpages.debian.org/jessie/dh-make/dh_make.8.en.html) sets up the baseline files we will use.  

```
dh_make  --createorig -e census-developers@googlegroups.com -s -y
```

This will create the [Required files under the debian directory](https://www.debian.org/doc/manuals/maint-guide/dreq.en.html)

5) Configure installer

The link and references above covers the various files and how to set them up.  If you just want the quick+dirty, preconfigured files, you can copy the ones spcific to this installer from here:

```
cp -R ../stage/debian/* debian/
```

The noteworthy files that we edited are:

- `debian/control`:  describes what this repo does
- `debian/compat`
- `debian/files`
- `debain/opencensusagent-install`: where to copy the installer files (in our case from `debian/src/opt/opencensus-agent/*` to `/opt/`)
- `debian/opencensusagent.service`: `systemd` configuration
- `debian/preinst`: preinstallation tasks to setup the user/group, etc
- `debian/postinst`: post installation tasks
- `debian/postrm`: remove the user/group and directories after the package is removed
- `debian/rules`: this is an important file:  this describes that we will install systemd and any overrides to systemd or installation to apply.
- `debian/source/include-binaries`:  the target binaries we need to package up (in our case, the opencensus binary and its config file)

and the actual binaries we will install (remember, we're not compiling with this deb file!)
- `debian/src/opt/opencensus-agent/bin/ocagent`
- `debian/src/opt/opencensus-agent/conf/oca.yaml`

6) Populate the opencensus binary

THis is done because i didn't include the agent in this git repo!

```bash
wget https://github.com/census-instrumentation/opencensus-service/releases/download/0.1.3/ocagent_linux \
  -O debian/src/opt/opencensus-agent/bin/ocagent
```

7) Build .deb file

Now we're finally ready to build:

```bash
dpkg-buildpackage -kcensus-developers@googlegroups.com
```
(remember, the password is `opencensus01`)

This will create the `.deb` candidate in a directory above the current (`../opencensusagent_0.1.2-1_all.deb`)

8) Test .deb file candidate

To test, we'll be setting up a GCP VM (so..yeah, i'm assuming you have access; otherwise, pick a debian system somewhere)

```bash
cd  ../
gcloud compute instances create oc-client1 --zone=us-central1-a
gcloud compute scp opencensusagent_0.1.2-1_all.deb oc-client1:/tmp/
gcloud compute ssh oc-client1
```


8) Install `deb`

SSH to the VM 

```
gcloud compute ssh oc-client1 --zone=us-central1-a
sudo su -
cd /tmp/

dpkg -i opencensusagent_0.1.2-1_all.deb 
    Selecting previously unselected package opencensusagent.
    (Reading database ... 34432 files and directories currently installed.)
    Preparing to unpack opencensusagent_0.1.2-1_all.deb ...
    Adding system user `ocagent' (UID 108) ...
    Adding new group `ocagent' (GID 112) ...
    Adding new user `ocagent' (UID 108) with group `ocagent' ...
    Not creating home directory `/home/ocagent'.
    Unpacking opencensusagent (0.1.2-1) ...
    Setting up opencensusagent (0.1.2-1) ...
    Created symlink /etc/systemd/system/multi-user.target.wants/opencensusagent.service → /lib/systemd/system/opencensusagent.service.
    Processing triggers for systemd (232-25+deb9u8) ...
```

9) Verify the `.deb`


```bash
$ service opencensusagent start

$ service opencensusagent status
  ● opencensusagent.service - Opencensus Agent
    Loaded: loaded (/lib/systemd/system/opencensusagent.service; enabled; vendor preset: enabled)
    Active: active (running) since Wed 2019-03-06 01:30:37 UTC; 1s ago
  Main PID: 894 (ocagent)
      Tasks: 7 (limit: 4915)
    CGroup: /system.slice/opencensusagent.service
            └─894 /opt/opencensus-agent/bin/ocagent -c /opt/opencensus-agent/conf/oca.yaml

  Mar 06 01:30:37 oc-client1 systemd[1]: Started Opencensus Agent.
  Mar 06 01:30:38 oc-client1 opencensusagent[894]: 2019/03/06 01:30:38 Running OpenCensus Trace and Metrics receivers as a gRPC service at "localhost:55678"
  Mar 06 01:30:38 oc-client1 opencensusagent[894]: 2019/03/06 01:30:38 Running zPages at ":55679"


$ ps -ef | grep ocagent
    root      18473      1  0 01:01 pts/0    00:00:00 /opt/opencensus-agent/bin/ocagent -c /opt/opencensus-agent/conf/oca.yaml
    root      18782      1  0 01:03 pts/0    00:00:00 grep ocagent

$ service opencensusagent stop
    [ ok ] Stopping opencensusagent:.

```

10) Uninstall the deb

```bash
$ dpkg -r opencensusagent
$ dpkg -P opencensusagent
```

## Create APT Repository

Ok now that we've verified the the `.deb`, lets package this in to an apt-repo

Back on your local vm 

11) Create a 'docroot' of the repo and copy the opencensus `.deb` files over

```bash
mkdir repo && cd repo/
cp ../opencensusagent_0.1.2-1_all.deb .
```

12) Create a default `conf/distribution` file under repo

`conf/distributions`  has the following content:

```
Origin: OpenCensus (https://opencensus.io/)
Label: OpenCensus
Codename: stretch
Architectures: amd64
Components: main
Description: OpenCensus Package Repo
SignWith: 0838A470
Pull: stretch

Origin: OpenCensus (https://opencensus.io/)
Label: OpenCensus
Codename: trusty
Architectures: amd64
Components: main
Description: OpenCensus Package Repo
SignWith: 0838A470
Pull: trusty
```


13) Generate the archive

```
$ reprepro  -Vb . includedeb stretch opencensusagent_0.1.2-1_all.deb

  opencensusagent_0.1.2_all.deb: component guessed as 'main'
  Created directory "./pool"
  Created directory "./pool/main"
  Created directory "./pool/main/o"
  Created directory "./pool/main/o/opencensusagent"
  Exporting indices...
  Created directory "./dists"
  Created directory "./dists/stretch"
  Created directory "./dists/stretch/main"
  Created directory "./dists/stretch/main/binary-amd64"
  Successfully created './dists/stretch/Release.gpg.new'
  Successfully created './dists/stretch/InRelease.new'
```

>> Actually, thats it; you've got the repo now all setup.  We just need to setup a webserver-distribution

Final step is remove the deb and prepare the archive so we can tranmit it to the apt repo we will just setup next

```
rm opencensusagent_0.1.2-1_all.deb
cd ../
tar cvf repo.tar
```

13) Setup Webserver distribution point

Now lets prepare the VM that will host the repo with a serving certificate (since we want to use https!)

- Create ssl cert for the `https` distribution

```
openssl req  -nodes -new -x509  -keyout server.key -out server.cert
```
(since this is testing; add any CN= value you want)


14) Create VM to host the repo

```
gcloud compute instances create repo --zone=us-central1-a
NAME  ZONE           MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
repo  us-central1-a  n1-standard-1               10.128.0.48  35.202.237.155  RUNNING
```

>> Note the internalIP `10.128.0.48` 


15) Copy the archive and certificate

```
gcloud compute scp repo.tar repo:/tmp/ --zone=us-central1-a
gcloud compute scp server* repo:/tmp/ --zone=us-central1-a
gcloud compute scp stage/certs/oc_public.key repo:/tmp/  --zone=us-central1-a
```

16) Configue nginx to service https

```
gcloud compute ssh repo
sudo su -
apt-get update 
apt-get install nginx -y
```

- Copy certs over:
```
cp /tmp/server.key /etc/nginx/
cp /tmp/server.cert /etc/nginx/
```


- configure nginx

```
vi /etc/nginx/sites-enabled/default
```


```
server {
    listen              443 ssl;
    listen              [::]:443 ssl;
    server_name         example2.com www.example2.com;
    root                /var/www/html;

    ssl_certificate     /etc/nginx/server.cert;
    ssl_certificate_key /etc/nginx/server.key;
    }
```

- copy public key and certificate to the docroot

```
cd /var/www/html
cp /tmp/oc_public.key .
cp /tmp/repo.tar .
tar xvf repo.tar 

service nginx restart
```


### End-to-End Test

Now to test the repo from a new client!

in a new shell, create the 'client'.  We want the

- setup some background libraries;tools
```
apt-get update
apt-get install -y apt-transport-https ca-certificates curl gnupg2 software-properties-common vim
```

- Download and allow apt to trust our public key

```
  curl -fsSLk https://10.128.0.48/oc_public.key | apt-key add -
  OK
```

Check the cert fingerprint matches

```
apt-key fingerprint 141D7911
  pub   rsa4096 2019-03-01 [SC]
        9A70 3639 96C3 EE27 9ABB  AF2E 0C74 009E 141D 7911
  uid           [ unknown] OpenCensus (https://opencensus.io) <census-developers@googlegroups.com>
```


- Install the repo

```
add-apt-repository "deb http://10.128.0.48/repo $(lsb_release -cs) main"
apt-get update
```

- Install the package!

```
apt-get install opencensusagent
```

- Verify
```
service opencensusagent start

service opencensusagent status
● opencensusagent.service - Opencensus Agent
   Loaded: loaded (/lib/systemd/system/opencensusagent.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2019-03-06 01:52:52 UTC; 33s ago
 Main PID: 27922 (ocagent)
    Tasks: 8 (limit: 4915)
   CGroup: /system.slice/opencensusagent.service
           └─27922 /opt/opencensus-agent/bin/ocagent -c /opt/opencensus-agent/conf/oca.yaml

Mar 06 01:52:52 oc-client1 systemd[1]: Started Opencensus Agent.
Mar 06 01:52:53 oc-client1 opencensusagent[27922]: 2019/03/06 01:52:53 Running OpenCensus Trace and Metrics receivers as a gRPC service at "localhost:55678"
Mar 06 01:52:53 oc-client1 opencensusagent[27922]: 2019/03/06 01:52:53 Running zPages at ":55679"
```

>> note, if you want to run the agent via commandline
```sudo -H -u ocagent bash -c '/opt/opencensus-agent/bin/ocagent -c /opt/opencensus-agent/conf/oca.yaml'```


- Uninstall

```
apt-get remove opencensusagent
apt-get purge opencensusagent
```

## Done

Ok, so i just did something that completely nothing now...just wrote this up since i did this from scratch; hopefully somone finds some of these step sueful
