# CREATE A LOCAL TRUSTED REGISTRY FOR PODMAN AND CREATE YOUR OWN QT DEV IMAGE

Note: This was created with much starting, restarting, gratuitous cursing, googling, banging, pacing back and forth and several decent pours of bourbon on a Fedora 42 Server linux distribution.

## Edit Container Config
This should be done on your server or machine where the registry will be located.

### Step
```bash
sudo vi /etc/containers/registries.conf
```

```bash
#Find this entry in the config and prepend your server ip and selected port for
# registry connections. Then below addd [[registry]] entry for that registry. There
# should be none if this is your first time and there should be a commented out one.

unqualified-search-registries = [ "registry.fedoraproject.org", "registry.access.redhat.com", "docker.io"]

# [[registry]]
```
save the file and exit. Your final conf file should have the same look as below except for your ip's and/or hostnames:

```bash
>unqualified-search-registries = [ "hal2000:5000", "192.168.1.3:5000", "registry.fedoraproject.org", "registry.access.redhat.com", "docker.io"]

[[registry]]
location = "192.168.1.3:5000"
insecure = false

[[registry]]
location = "hal2000:5000"
insecure = false
```

NOTE: 
The following two sections are optional.  These configuration changes change the location of where container layers are stored or temporarily stored during podman transactions.  By default it is in a /var/lib/XXXX which is usually tied to root in most installations.  You will know better if you custom config your installation.  But I normally give the bare minimum plus extra to root for system only plus future updates.  I typically install things in alternate locations for backup or isolation from system.  ( ie: I don’t want to run out of disk space due to root “/” filling up. )

## Edit /usr/share/containers/containers.conf (Optional)

Step
```bash
>sudo vi /usr/share/containers/containers.conf
```
change
```
image_copy_tmp_dir="/home/datadisk1/var/tmp"
```
What it does:
This is the default location for storing temporary image content.
Note: 
Any changes to configuration may appear in “sudo podman info”, but they most probably will not be used until you restart podman.
```bash
> pkill podman
> sudo systemctl restart podman.service
```
## Edit /usr/share/containers/storage.conf (Optional)
Step

```bash
>sudo vi /usr/share/containers/containers.conf
```
change
from:
```bash
runroot = "/run/containers/storage" 
```
to:
```bash
runroot = "/home/datadisk1/var/tmp"
```
What this does:
Changes the temporary storage location

Step
change
from:
```bash
graphroot = "/var/lib/containers/storage" 
```
to:
```bash
graphroot = "/home/datadisk1/var/lib/containers/storage" 
```

What this does:
Changes primary read/write location of container storage. Because this is a fedora distribution or because I make sure SELINUX is active, you must 
ensure the labeling matches the default locations labels with the 
following commands: 
```bash
>semanage fcontext -a -e /var/lib/containers/storage /NEWSTORAGEPATH 
>restorecon -R -v /NEWSTORAGEPATH
```
Step
change 
from:
```bash
rootless_storage_path = "$HOME/.local/share/containers/storage" 
```
to:
```bash
rootless_storage_path = "/home/datadisk1/var/lib/containers/rootless/storage"
```
What this does:
Changes the storage path for rootless users.

Step

change
from:
```bash
  # AdditionalImageStores is used to pass paths to additional Read/Only image stores 
  # Must be comma separated list. 
  additionalimagestores = [ 
  "/usr/lib/containers/storage",
  ]
```
to:
```bash
  # AdditionalImageStores is used to pass paths to additional Read/Only image stores 
  # Must be comma separated list. 
  additionalimagestores = [ 
  "/home/datadisk1/var/lib/containers/readonly/storage", 
  ] 
```
What this Does:
This adds additional places of storage to search for read only image.  This would be shares or network shares thus allowing the images to be accessed over the network. Again the default is putting (depending on your drive layout ) the data in “/” which I don’t want. I made a specific directory for this, and if necessary can be used to allow images to be accessed remotely instead of being pulled.  However most of the time most people don’t use this feature so its not considered necessary to have a working podman installation.

Note: 
Any changes to configuration may appear in “sudo podman info”, but they most probably will not be used until you restart podman.
```bash
> pkill podman
> sudo systemctl restart podman.service
```
           
## Create container volumes for trust and security and data
Step
```bash
>sudo mkdir -p /opt/registry/{auth,certs,data}
```
What it does:
On your server or registry location, create the certs directory where certificates will be generated and pointed to by a container volume. Create the auth directory where the htpasswd authorization file will be kept and pointed to by a container volume.  Create the data directory where container volatile data will be stored via a container volume.

## Create the self signed certificates needed for connection trust
Now we begin the process of creating the certificates. Self signed certificates already act as a self signed authority.  However what instruction set would be complete unless you follow through creating your very own self signed Certificate Authority (CA) for your local network.

Step
```bash
>cd /opt/registry/certs
>sudo vi san.cnf
```

```bash
Contents of file: san.cnf
[req]
default_bits = 4096
distinguished_name = req_distinguished_name
req_extensions = req_ext
x509_extensions = v3_req
prompt = no

[req_distinguished_name]
countryName = US
stateOrProvinceName = New Jersey
localityName = Mendham
commonName = 192.168.1.3: Self-signed certificate

[req_ext]
subjectAltName = @alt_names

[v3_req]
subjectAltName = @alt_names

[alt_names]
IP.1 = 192.168.1.3
IP.2 = 127.0.0.1
DNS = hal2000
```

What it does:
The most important aspect of this file is to setup the IP SANs. These are the Subject alternative names.  You can use either hostnames using the DNS.1 … DNS.n entries or in this case IP addresses using the IP.1 … IP.n entries.

Save file then exit.

## GENERATE YOUR OWN CERTIFICATE AUTHORITY(CA):
Step
```bash
>cd /opt/registry/certs
>sudo openssl genrsa -des3 -out rootCA.key 4096
```

What it does:
Generates a private key for your root CA (Certificate Authority). It is using 4096 encryption bits

Step
```bash
>sudo openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 5000 -out rootCA.crt
```

What it does:
Generate the self-signed root CA certificate using private key from previous step. This is your self signed Certificate Authority. -days represents how many days your certificate will be valid before expiration.

## GENERATE YOUR SELF SIGNED CERTIFICATE, SIGNED BY YOUR CA 
Step
```bash
>sudo openssl genrsa -out /opt/registry/certs/hal2000.key 4096
```

What this does:
Generates a private key for your registry server certificate. It is using 4096 as the number of encryption bits. 

Step
```bash
>sudo openssl req -new -sha256 -key /opt/registry/certs/hal2000.key -out /opt/registry/certs/hal2000.csr --config /opt/registry/certs/san.cnf
```

What it does:
Generate the Certificate Signing Request for the server certificate using the private key created from previous step.  --config instructs openssl to use our san.cnf file which we created in the beginning to specify IP SANs

Step
```bash
>sudo openssl x509 -req -in /opt/registry/certs/hal2000.csr -CA /opt/registry/certs/rootCA.crt -CAkey /opt/registry/certs/rootCA.key -CAcreateserial -out hal2000.crt -days 5000 -sha256 -extfile san.cnf -extensions v3_req
```

What it does:
Generate the signed server certificate. Openssl will take our Certificate Signing request (.csr file ), Our Certificate Authority certificate and our Certificate Authority private key.  This will produce a serial file and our signed certificate for our registry.  --days specifies the number of days the certificate should be valid.  -extfile instructs openssl to use our san.cnf file which we created to specify our IP SANs.  -extensions v3_req [VERY IMPORTANT] I cant tell you how many times I wanted to bang my head into the monitor because the IP SANs were not being put into the certificate.  By specifying the san.cnf file section v3_req it will post the data correctly. This is always missing from other tutorials and instructions.

You can verify your certificate and the information inside it as follows:

Step
```bash
>openssl x509 -noout -text -in hal2000.crt
```

What it does:
This will read your certificate and print out all the sections as text ( without decoding the key of course )

## PUT CA CERTIFICATE AND REGISTRY CERTIFICATE INTO TRUST
Step
```bash
>sudo cp /opt/registry/certs/rootCA.crt /etc/pki/ca-trust/source/anchors/.
>sudo cp /opt/registry/certs/hal2000.crt /etc/pki/ca-trust/source/anchors/.
>sudo update-ca-trust
```

What it does:
You’ve created your certificates, now they need to be added into your systems trusted certificate authority so they can be used by applications that rely on them. 

You can verify that your certificate is in the trust
1) If you used DNS in the sans.cnf file
>trust list | grep -i <hostname>
or
2) If you used IP in the sans.cnf file.
> trust list | grep -i <ip>

## Create Registry adding basic authentication
Step
>cd /opt/registry/auth
>sudo podman run --rm --entrypoint htpasswd registry:2.7.0 -bBn USERNAME "PASSWORD"

What this does:
We are telling podman to run the image from docker.io that represents a public registry image.  This will pull registry:2.7.0:latest from docker.io, store it locally, then run it.  It will create the htpasswd file in our current directory so make sure you are in /opt/registry/auth. The entry in htpasswd is created using USERNAME and “PASSWORD”.  I use enclosing quotes when the password contains special characters.

Step
```
>sudo podman run -d --name registry -p 5000:5000 \
-v /opt/registry/data:/var/lib/registry:z \
-v /opt/registry/auth:/auth:ro,z \
-e "REGISTRY_AUTH=htpasswd" \
-e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
-e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
-v /opt/registry/certs:/certs:z \
-e "REGISTRY_HTTP_TLS_CERTIFICATE=/certs/hal2000.crt" \
-e "REGISTRY_HTTP_TLS_KEY=/certs/hal2000.key" \
-e REGISTRY_COMPATIBILITY_SCHEMA1_ENABLED=true \
--replace \
docker.io/library/registry:latest
```
What it does:
This reruns the registry server with data volumes and env vars and replaces previously running registry.
--name	will give the registry container the specified name. You can refer to 			the registry by this name.
-p		specifies the mapping of host port to the container port. 
-v		specifies the host directory to map as a volume to a directory in the container.
-e		specifies an environment variable to be set in the container.
--replace	replace the currently running image if any.
docker.io/library/registry:latest is the name of the latest local image as downloaded.

Currently you have your own registry container running that can only be connected to by clients with the proper certificates and password as specified by USERID and password when creating the htpasswd entry.

Step
>sudo podman login -u USERNAME 192.168.1.3:5000

What it does:
You can verify on your server or registry location that you can login to the exposed registry ip:port.  You will need to make additional entries for users and their passwords (enjoy).


## Allow clients access to remote (Your Local) Registry
Note: This was created with much starting, restarting, gratuitous cursing, googling, banging, pacing back and forth and several decent pours of bourbon on a Fedora 42 Workstation linux distribution. 
                   
## Edit registries.conf on client machine
On the remote machine on your network
>sudo vi /etc/containers/registries.config

The entries here represent the server/location of your registry. If your registry and this machine are one and the same you don’t need to do anything further. You can skip the next sections of setup and move on to usage.

unqualified-search-registries = ["hal2000:5000", "192.168.1.3:5000", "registry.fedoraproject.org", "registry.access.redhat.com", "docker.io"]

[[registry]]
location = "192.168.1.3:5000"
insecure = false

[[registry]]
location = "hal2000:5000"
insecure = false


NOTE: 
The following two sections are optional.  These configuration changes change the location of where container layers are stored or temporarily stored during podman transactions.  By default it  is usually tied to a location in “/” (root) in most installations.  You will know better if you custom config your installation.  But I normally give the bare minimum plus extra to root for system only plus future updates.  I typically install things in alternate locations for backup or isolation from the os system. 

## Edit /usr/share/containers/storage.conf (Optional)
Step
```bash
>sudo vi /usr/share/containers/containers.conf
```

change
from:
```
runroot = "/run/containers/storage" 
```
to:
```bash
runroot = "/home/mbudwick/containertemp"
```

What this does:
Changes the temporary storage location

change
from:
```bash
graphroot = "/var/lib/containers/storage" 
```
to:
```bash
graphroot = "/home/mbudwick/containerstorage" 
```
What this does:
Changes primary read/write location of container storage. Because this is a fedora distribution or because I make sure SELINUX is active, you must 
ensure the labeling matches the default locations labels with the 
following commands: 
```bash
>semanage fcontext -a -e /var/lib/containers/storage /NEWSTORAGEPATH 
>restorecon -R -v /NEWSTORAGEPATH
```
This is noted in the config and skipping this step will result in sadness.

Note: 
Any changes to configuration may appear in “sudo podman info”, but they most probably will not be used until you restart podman.
```bash
> pkill podman
> sudo systemctl restart podman.service
```
## Install Certificates on the client machine

Step
```bash
>sudo mkdir /etc/containers/certs.d/<server ip>:port
```
example:
```bash
>sudo mkdir /etc/containers/certs.d/192.168.1.3:5000
```
What this does:
This creates the certificate location in the ./certs.d directory for the appropriate registry (server)

Step
```bash
>sudo scp -v user@<server ip>:/opt/registry/certs/hal2000.crt /etc/containers/certs.d/<server ip>:port/ca.crt
>sudo scp -v user@<server ip>:/opt/registry/certs/rootCA.crt /etc/containers/certs.d/<server ip>:port/rootCA.crt
```

What this does:
This is an ssh file copy utility to copy the certificate files from server to proper client machine destination directories. Note the server certificate destination name should be ca.crt. Feel free to use your own methods to get the files there if you don’t have scp or cant use it.  This is just one of many I used.

Step
```bash
>sudo cp /etc/containers/certs.d/192.168.1.3:5000/rootCA.crt /etc/pki/ca-trust/source/anchors/.
>sudo cp /etc/containers/certs.d/192.168.1.3:5000/hal2000.crt /etc/pki/ca-trust/source/anchors/.
>sudo update-ca-trust
```

What it does:
You’ve copied your certificates from server to this client, now they need to be added into this systems trusted certificate authority so they can be used by applications that rely on them. 

You can verify that your certificate is in the trust
1) If you used DNS in the sans.cnf file
```bash
>trust list | grep -i <hostname>
```
or
2) If you used IP in the sans.cnf file.
```bash
> trust list | grep -i <ip>
```
verify successful installation with :
```bash
>sudo podman login -u USERNAME <server ip>:port
```

If it still doesn’t work, feel free to re examine the steps you did ad nauseum.
And don’t forget the bourbon.

Currently you have a registry running on server and can access it from a client to pull your localized images.  But pretty useless right now because there are no images.

# LETS CREATE OUR OWN IMAGE FOR OUR REGISTRY
As our example image we will create our own Qt Development installation image so that we can maintain a fixed version for development that can be re-used again for whatever purpose without being tainted by version changes, library name changes or whatever over time.  This installation will not have to be compiled.  We will base it off a Qt community edition version installed locally. Download the Qt community edition installation (don’t forget to donate) and install into a generic directory that can be dropped into any system.  For my reason and purposes I chose /home/datadisk1/ContainerSources/Qt.  Note datadisk1 is not a user, its just part of the path. This is because I will use the same Qt installation path in the container.  Dont forget the Qt installation has many files with hard coded paths, so you don’t want to break the Qt installation by having it installed in one path location and then copying it to a different path location. You can choose /opt/Qt or any other path that would be easy to copy to the container.  If you do choose your own path don’t forget to edit the Dockerfile below accordingly.

## Dockerfile

The beauty of podman is that it allows us to have our own local registry while still using the same utility as docker.  So we will start with a Docker file.

On Server create a Dockerfile or name it as you wish.  Remember if you create your own filename, to specify this filename when running podman build command.  Run “podman build -–help” for more info

Note: (Repeat)the installation of qt was in an a system/user agnostic directory: /home/datadisk1/ContainerSources.   There is no datadisk1 user.  Its just a directory that can be replicated in the container without affecting Qt’s installation paths.

Step
>vi Dockerfile
```bash
# make sure to run podman build ... in same directory as Dockerfile 
# to further lock your distribution into a specific version you can remove the :latest tag # and specify a version.
FROM registry.fedoraproject.org/fedora:latest 

# mimic the same dir struct from installed version 
RUN mkdir -p /home/datadisk1/ContainerSources/Qt 

# add non root user 
ARG UID 
ARG GID 

RUN groupadd qtdev 
RUN adduser --uid $UID --gid $GID -G qtdev mbudwick 

# Set the working directory 
WORKDIR /home/datadisk1/ContainerSources/Qt 

# Copy the Qt directory into the container. Note we are working in a directory above Qt 
COPY ./Qt /home/datadisk1/ContainerSources/Qt 
RUN chown -R mbudwick:qtdev /home/datadisk1/ContainerSources/Qt 

RUN mkdir -p /home/mbudwick/.cache/qt-unified-linux-online 

# Install build tools and Qt dependencies 
RUN dnf update -y 
RUN dnf repolist 
RUN dnf install -y libX11* 
RUN dnf install -y libxkb* 
RUN dnf install -y libxcb* 
RUN dnf install -y libwayland-egl 
RUN dnf install -y xcb-util-cursor 
RUN dnf install -y xcb-util-cursor-devel 
RUN dnf install -y xcb-util-keysyms 
RUN dnf install -y xcb-util-keysyms-devel 
RUN dnf install -y xcb-util-wm 
RUN dnf install -y xcb-util-wm-devel 
RUN dnf install -y libsecret 
RUN dnf install -y libsecret-devel 
RUN dnf install -y wayland-devel 
RUN dnf install -y libwayland-client 
RUN dnf install -y libwayland-server 
RUN dnf install -y libwayland-egl 
RUN dnf install -y freetype 
RUN dnf install -y libxkbcommon 
RUN dnf install -y mesa-libGL-devel 
RUN dnf install -y mesa-libGLU-devel 
RUN dnf install -y mesa-libEGL-devel 
RUN dnf install -y fontconfig 
RUN dnf install -y libXrandr 
RUN dnf install -y dbus-libs 
RUN dnf install -y dbus-devel 
RUN dnf install -y libXext-devel 
RUN dnf install -y xorg-x11-server-Xorg 
RUN dnf install -y xorg-x11-xauth 
RUN dnf install -y xclock 
RUN dnf install -y gcc 
RUN dnf install -y gcc-c++ 
RUN dnf install -y python3 
RUN dnf install -y boost 
RUN dnf install -y boost-devel 

ENV LANG=C.UTF-8 
ENV LC_ALL=C.UTF-8 
ARG QT_VERSION=6.9.0 
ENV QT_VERSION=${QT_VERSION} 
ENV QT6DIR=/home/datadisk1/ContainerSources/Qt/6.9.0/gcc_64 
ENV QT_QPA_PLATFORM=xcb 
ENV QT_QPA_PLATFORM_PLUGIN_PATH=/home/datadisk1/ContainerSources/Qt/6.9.0/gcc_64/plugins/platforms 
ENV QT_DEBUG_PLUGINS=1 

ENV QT_DIR=/home/datadisk1/ContainerSources/Qt/6.9.0/gcc_64 
ENV QT6PREFIX=/home/datadisk1/ContainerSources/Qt/6.9.0/gcc_64 
ENV QT_HOME=/home/datadisk1/ContainerSources/Qt/6.9.0/gcc_64 
ENV PATH=$QT_HOME/bin:$PATH 
ENV export LD_LIBRARY_PATH=$QT_HOME/lib:$LD_LIBRARY_PATH 
ENV export LD_LIBRARY_PATH=/home/datadisk1/ContainerSources/Qt/Tools/QtCreator/lib/Qt/lib:$LD_LIBRARY_PATH 
ENV QT_PLUGIN_PATH=/home/datadisk1/ContainerSources/Qt/6.9.0/gcc_64/plugins 

# Set environment variables for X11 forwarding  
#ENV DISPLAY=$DISPLAY 

ENV export XDG_RUNTIME_DIR=/run/user/$(id -u) 
ENV QT_X11_NO_MITSHM=1 
# Just in case
EXPOSE 80 

#RUN echo "root:root" | chpasswd 

# this is a fix for version of libxcb to be found
RUN ln -s /lib64/libxcb-cursor.so.0 /lib64/libxcb-cursor0.so.0 

RUN echo 'mbudwick ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers 
USER mbudwick 

WORKDIR /home/homedatadisk1/ContainerSources/Qt 

# Command to run when the container starts 
CMD ["/home/datadisk1/ContainerSources/Qt/Tools/QtCreator/bin/qtcreator"]
```

Save file and exit.

## Building Image from Dockerfile
Lets make a helper script thats sets a couple environment variables and calls podman to build our image from above Dockerfile.

Step
>vi buildQtDockerfile.sh
```bash
#!/bin/sh 
export HOST_UID=$(id -u) 
export HOST_GID=$(id -g) 
# This next line is only if you did the optional config edits.  This is supposed to
# ensure TMPDIR is set but never did work for me until I edited config and restarted # podman. The –-tmdir flag I think did have an effect. Left in for example to others.
export TMPDIR=/home/datadisk1/var/tmp  
sudo podman build --tmpdir=/home/datadisk1/var/tmp --build-arg UID=$HOST_UID --build-arg GID=$HOST_GID -t myqtdevtainer .
```

Save file and exit. We are specifying name of image in -t argument.  Name it what you wish.

Step
```bash
>chmod +x ./buildDockerfile.sh
```

What this does:
Makes the new script file executable.

Step
```bash
>./buildQtDockerfile.sh
```
What this does:
Basically starts the creation of our image based on the instructions in the Dockerfile.  You can go to Columbia for some coffee.  It should be a while.

## Tag new image and push to our fancy new Registry
Step
```bash
>sudo podman tag localhost/myqtdevtainer 192.168.1.3:5000/myqtdevtainer:6.9.0
>sudo podman tag localhost/myqtdevtainer 192.168.1.3:5000/myqtdevtainer:latest
```

What this does:
So far the new image was created with the name myqtdevtainer.  You can use tags to give the image a proper version.  We use myqtdevtainer:6.9.0 because that is the current version of Qt that is being used.  This makes it easier to choose which image you need. The line after that makes this version also the latest version.  So you can either specify the version, or leave it off and it will by default use the image tagged as latest.

Step
```bash
>sudo podman image push --creds mbudwick 192.168.1.3:5000/myqtdevtainer:latest 192.168.1.3:5000/registry/myqtdevtainer
```
What this does:
This will push the image we just tagged as latest to our new registry.  --creds allows us to specify our userid that is authorized and you will need to use the password setup on the server for the htpasswd entry.  If no creds are specified you will get a connection refused.  

# TEST INSTALLATION ON CLIENT
## Pull Image from Server
You will need to go to a remote or client machine different from server to test these commands.

Step
```bash
>sudo podman image pull --creds mbudwick 192.168.1.3:5000/registry/myqtdevtainer:latest
```

What this does:
This will pull the specified image off server registry to local storage.  This way you can run it without being on the same network all the time.  Again –creds specify authorized user

## Run image in local container

Step
```bash
>sudo podman run -it --rm --network=host --privileged -v /tmp/.X11-unix:/tmp/.X11-unix -v $XAUTHORITY:$XAUTHORITY -v ~/Development:/home/mbudwick/Development -e DISPLAY=$DISPLAY --security-opt label=type:container_runtime_t 192.168.1.3:5000/registry/myqtdevtainer:latest
```
What this does:
Runs your image. --network says to use the hosts network. -v ~Development:/home/<user>/Development specifies the development directory volume that is mapped in the container to a directory name under the authorized user.  This can be local files or even files managed by source control.
