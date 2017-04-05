# Building Triton

## Prerequisites

 * we assume you have npm/node installed on your workstation
 * we assume you have git installed on your workstation
 * we assume you have json (npm install -g json) installed on your workstation
 * we assume you understand the basics of Triton, if not please start with [the
   Triton DataCenter README](https://github.com/joyent/triton#readme)
 * we assume you have your SSH keys loaded in your ssh-agent when connecting
   to build zones via SSH.

## Decisions to Make

To build Triton components you to decide first make a couple choices:

 * which components are you going to build?
 * where are you going to build components?

### Components

If you are building any of the following components:

 * cns
 * manta-manatee
 * sdc-manatee
 * electric-moray
 * platform

you will need an sdc-multiarch 13.3.1 build zone. For the following components,
you will need an sdc-minimal-multiarch-lts 15.4.1 zone:

 * cloudapi
 * cmon
 * docker
 * imgapi
 * nfsserver
 * papi
 * vmapi
 * volapi

For any other components you will need an sdc-smartos 1.6.3 zone. If you want
to build *all* components, you'll need all three.  The targets.json file
(created by targets.json.in) in MG contain the definitive image UUIDs.
The ./tools/target-\*.sh scripts in MG (see "Clone MG (mountain-gorilla)") use
this file to output the list of components for each respective version.

### Where to build

Currently, the easiest method for building Triton is to do so on a local
Triton or CoaL install.  Easily building the images in the Triton Public Cloud
(TPC) depends on the work described in
[RFD 46](https://github.com/joyent/rfd/blob/mastert/rfc/0046/README.md) to be
complete.

NOTE: Even if you do not use TPC and output your builds to Manta, you will still
need a Manta account (either local install or TPC) in order to do a build.
The build process will store built components into Manta and will utilize Manta
to retrieve dependent components during the build process.

## Setting up your workspace for driving builds

### Clone MG (mountain-gorilla)

On your local workstation (tested with OS X, but should work elsewhere) you can
run:

```
git clone git@github.com:joyent/mountain-gorilla.git MG && cd MG
```

Most of the rest of these instructions can be performed from within this
directory.

After cloning, you should run:

```
npm install
```

in this directory to setup the dependencies. Once you've done that, you can also
run:

```
export PATH=${PATH}:$(pwd)/node_modules/smartdc/bin
```

### Setup your environment variables

This is one of the most critical steps. The environment variables define which
Triton/manta target/credentials will be used for the rest of setup. You must set
at least:

```
export MANTA_USER=<USER>
export MANTA_KEY_ID=<KEY>
export MANTA_URL=https://us-east.manta.joyent.com
export SDC_ACCOUNT=<USER>
export SDC_KEY_ID=<KEY>
export SDC_URL=https://us-east-1.api.joyentcloud.com
```

where <USER> is the name of the TPC/Triton user you want to build with, and
<KEY> is the SSH fingerprint of the SSH key that you've added for your user.

If you're using CoaL and using the default self-signed certificates for cloudapi
you will also want to:

```
export TRITON_TLS_INSECURE=1
```

otherwise you'll get errors like:

```
sdc-listpackages: error: undefined
```

It's possible to use different <USER> values for MANTA_USER and SDC_ACCOUNT if
you've pointed these at different Triton standups. In that case zones will be
created using the SDC_ACCOUNT credentials and any files pulled from / pushed to
Manta will be done using the MANTA_USER's credentials.

Be sure to change the SDC_URL and MANTA_URL above to match your local cloudapi
and manta respectively.

NOTE: if your Triton install is not yet setup, you need to set SDC_URL *after*
setting up cloudapi in the next section.


## Setting up Triton for builds

This section assumes that you have a local Triton/CoaL setup and have access to
the global zone.

### Setting up cloudapi

If you are using CoaL you won't have cloudapi by default. To add it you can
run the following from within the MG directory on your workstation:

```
./tools/setup-cloudapi.sh root@<HN_GZ_IP>
```

where <HN_GZ_IP> is the IP of the GZ of your headnode. This script will login
and create the cloudapi zone for you.

If you're using CoaL and don't have any CNs attached to your headnode, you
will also want to login to the GZ of your CoaL headnode and run:

```
/zones/$(vmadm lookup -1 tags.smartdc_role=cloudapi)/root/opt/smartdc/cloudapi/tools/coal-setup.sh
```

in order that you can provision using cloudapi on your headnode.


### Setting up imgapi

In order to perform builds you need to add an external interface to the imgapi
zone. You also need to setup firewall rules (default rules do not allow
connections on the external interface) that allow your build zone(s) to connect
to the external interface of imgapi.

Refer to the SDC documentation for details on how to perform the required steps
here.

Firewall rules can also be setup after the build zone(s) are created, but before
the first build.

Record the external IP address for imgapi. You'll need this later to set
SDC_IMGAPI_URL.

For your convienence, here are commands for the previous steps if you are
running in COAL:

```
# Add an external nic
$ /usbkey/scripts/add_external_nic.sh $(vmadm lookup alias=~imgapi)
# Make sure the job finished successfully
$ sdc-workflow /jobs/405b26f1-0f6a-4118-aacb-0d89fd777a36 | json -Ha execution
succeeded
# Find the external imgapi ip address
$ sdc-vmapi /vms/$(vmadm lookup alias=~imgapi) | json -H nics | \
    json -ac 'this.nic_tag === "external"' ip
10.88.88.3
```

If you're using your own Triton install and do not have imgapi connected to a
Manta (eg.  you're using CoaL) you'll also need to run the following from the
GZ of the headnode:

```
echo '{"metadata": {"IMGAPI_ALLOW_LOCAL_CREATE_IMAGE_FROM_VM": true}}' \
  | sapiadm update $(sdc-sapi /services?name=imgapi | json -H 0.uuid)
```

### Importing the images

If you are using CoaL the sdc-smartos and sdc-multiarch images should already
be imported. If for some reason your setup does *not* have these images, you'll
need to import them. Follow the SDC documentation on importing images. The
images you need are:

 * fd2cc906-8938-11e3-beab-4359c665ac99 / sdc-smartos 1.6.3
 * b4bdc598-8939-11e3-bea4-8341f6861379 / sdc-multiarch 13.3.1
 * 18b094b0-eb01-11e5-80c1-175dac7ddf02 / sdc-minimal-multiarch-lts 15.4.1

### Adding build packages

The build zone needs a package sufficiently large enough to allow the builds
to complete.  If using a CoaL install, the easiest way is to import the
sample packages and use the sample-4G package.  To import the sample packages,
run the following command from the headnode:

```
sdcadm post-setup dev-sample-data
```

Optionally, if present, the sdc_4096 package will also work fine, however
the Triton user doing the build must be given access to the package before
it can be used to create an instance.

## Setting up the build environment(s)

Based on the choices you made earlier (see "Decisions to Make" section) you
should know which build zones you will need. This section will guide you through
the creation of the required zones.

### Common steps to creating any build zone

Before you continue, ensure that whatever user you're going to use (whether
'admin' or other user in your local Triton/CoaL) has your SSH keys added to it.
This is important as these instructions will have you running triton commands
and manta commands which will need these credentials.

If you don't do this you'll see errors like:

```
sdc-listpackages: error (InvalidCredentials): Invalid key d5:19:78:bb:d8:f5:ba:cd:6b:40:96:3f:5a:23:59:a9
```

### Creating a sdc-smartos 1.6.3 build zone

To create a sdc-smartos 1.6.3 zone you'll want to run (using the sample-4G
package, replace with your own value if using a different package):

```
triton instance create \
    --name "build-1.6.3" \
    -w sdc-smartos@1.6.3 sample-4G
```

This will also wait until the instance has finished provisioning before
returning.  Alternatively, you can run the following command to check on
the instance status:

```
triton instance get build-1.6.3 | json state
```

substituting the instance name given during instance creation.  Once the
instance is running, you can find it's IP using:

```
triton ip build-1.6.3
```

(substituting the name you gave the instance during creation).

At this point you'll want to take the IP address which is public (in the case
there are more than one) and fill that in as <BUILD_ZONE_IP> in the section
"Preparing build zones for builds" below.


### Creating an sdc-multiarch 13.3.1 build zone

To create a sdc-multiarch 13.3.1 build zone, you should follow the steps for
creating a sdc-smartos 1.6.3 build zone with the exception that instead of
sdc-smartos@1.6.3 you should use "sdc-multiarch@13.3.1" as well as a different
name.  For example:

```
triton instance create \
  --name "build-13.3.1" \
  -w sdc-multiarch@13.3.1 sample-4G
```

### Creating an sdc-minimal-multiarch-lts 15.4.1 build zone

Similarly, use 'sdc-minimal-multiarch-lts@15.4.1' and a new name.  For example:

```
triton instance create \
  --name "build-15.4.1" \
  -w sdc-minimal-multiarch-lts@15.4.1 sample-4G
```

### Preparing build zone(s) for builds

For each build zone you want to follow the same set of instructions. First
from your local MG repo you want to do:

```
./tools/setup-remote-build-zone.sh root@<BUILD_ZONE_IP>
```

This will produce some output as it logs into your zone, installs some packages
and generally gets it ready for you to login and start some builds.

### Cloning MG in your build zone

For each build zone you want to clone MG before you start
building. So SSH to the build zone, then run:

```
ssh -A root@<BUILD_ZONE_IP> # Use the -A to forward your SSH agent
git clone git@github.com:joyent/mountain-gorilla.git MG && cd MG
```

### Add additional environment variables

If you're building in TPC, you can skip this step. If you're building in a local
SDC/CoaL setup, you'll probably also need to also set the following at this
point:

```
export SDC_LOCAL_BUILD=1
export SDC_IMAGE_PACKAGE=sample-4G
export SDC_TESTING=1
export SDC_IMGAPI_URL=https://10.88.0.15
```

You'll also need to ensure these variables are set at the time of each build.

where:

 * SDC_LOCAL_BUILD tells MG that you don't want the build creating zones in JPC
   or pushing files to JPC's Manta as part of the build process.
 * SDC_IMAGE_PACKAGE is the name of the package you want to use for the build
   zones. CoaL ships with an sdc_4096 package which should work, or the
   sample-4G package if imported will also work.
 * SDC_TESTING allows the node-smartdc tools to work even when you've got a
   self-signed SSL certificate.
 * SDC_IMGAPI_URL should be set to https://<IP> where <IP> is the external IP
   you added to imgapi (remembering to add the firewall rules if you have not
   already)

## Building

The following commands should be run in the MG directory in the appropriate
build zone for the target you're building. They should also be run with all the
environment variables described earlier set.

By default, the build process will attempt to fetch the dependencies from
/${MANTA_USER}/stor/builds.  The `-d` and `-D` options can be used to specify
an alternate Manta account and location (respectively).

### Option 1: build a single target, taking dependencies from Joyent_Dev

Ensure you've set the appropriate environment variables, especially:

 * SDC_LOCAL_BUILD if you're building against your own SDC/CoaL
 * SDC_URL set to the proper cloudapi

Then to build, run the following in your MG directory in your build zone:

```
TARG=<build>; ./configure -t ${TARG} -d Joyent_Dev -D /stor/builds \
    -O /stor/whatever/builds && make ${TARG}_upload_manta
```

if we use 'assets' for the build for example:

```
TARG=assets; ./configure -t ${TARG} -d Joyent_Dev -D /stor/builds \
    -O /stor/whatever/builds && make ${TARG}_upload_manta
```

which will:

 * download dependencies from /Joyent_Dev/stor/builds
 * create a tarball of the assets bits + dependencies
 * create a SmartOS VM in the Triton install indicated in SDC_URL (using
   cloudapi)
 * install the tarball of bits into the VM
 * create an image from the VM, sending to Manta
 * download the image from Manta modify the manifest
 * push the build back to manta in ${MANTA_USER}/stor/whatever/builds/assets

Note: most packages include a required platform version for building.  This
is used so that images are built on the oldest supported (at build time)
platform.  The `-P` flag can be added to the `./configure` command to bypass
this requirement.

It is also possible to build a component on an image other than the one
indicated in the MG ./targets.json file (such as for testing).  To do so,
add `<TARGET>_IMAGE_UUID=<alternate image UUID>` as a parameter to make.
For example (including ignoring the platform version check):

```
TARG=<something>; ./configure -t ${TARG} -d Joyent_Dev -D /stor/builds \
    -O /stor/whatever/builds -P && make ${TARG}_upload_manta \
    ${TARG}_IMAGE_UUID=ede31770-e19c-11e5-bb6e-3b7de3cca9ce
```

### Option 2: build a single target, taking dependencies from Joyent_Dev but not uploading results

To *not* upload results to Manta, follow the same procedure as in "Option 1" but
change the make target from:

```
make ${TARG}_upload_manta
```

to:

```
make ${TARG}
```

The result will then be in the bits/ directory instead of going to Manta.

### Option 3: build all targets from scratch

If you want to ensure you've built every bit that you're using, you'll want to
do your builds in order and send them to a fresh Manta area. There's a tool
in MG's tools directory that will help you build in the correct order. For
example, assuming I'm running as Manta user "joyager" and I want to create a
full set of builds and then use that to build a new headnode image, I'd start
in my 1.6.3 zone and run:

```
(set -o errexit
    for TARG in $(./tools/targets-1.6.3.sh); do
        ./configure -t ${TARG} -d joyager -D /stor/whatever/builds \
            -O /stor/whatever/builds && make ${TARG}_upload_manta
    done
)
```

which will build all the dependencies first then the 1.6.3-built images.
Uploading to /joyager/stor/whatever/builds (which was empty when we started) and
taking dependencies only from that directory.

Once this is complete, we can run the same command just with:

```
./tools/targets-13.3.1.sh
```

and then

```
./tools/targets-15.4.1.sh
```

instead of:

```
./tools/targets-1.6.3.sh
```

generating the target list. The only target that currently cannot be built
this way which is required for building a new headnode image is the platform
target. The next section will deal with that.

### Building sdc-headnode without using manta at all

Assuming you've set all the environment variables and setup both build zones
you need (including the modifications in the "Building the platform image"
section) and setup your cloudapi and so forth, you can use the instructions in
this section to build everything locally without Manta.

Start on the 1.6.3 build zone and run:

```
mkdir /root/MY_BITS
export LOCAL_BITS_DIR=/root/MY_BITS
```

Then cd to your MG workspace on this zone and run:

```
(set -o errexit; for TARG in $(tools/targets-1.6.3.sh); do \
    ./configure -t ${TARG} -d joyager -D /stor/donotuse/builds \
        -O /stor/donotuse/builds && make ${TARG}_local_bits_dir; done)
```

This will take a while. Once it completes, create the /root/MY_BITS directory
on your 13.3.1 build zone and set the LOCAL_BITS_DIR variable:

```
mkdir /root/MY_BITS
export LOCAL_BITS_DIR=/root/MY_BITS
```

Now transfer all the bits from your 1.6.3 build zone to this 13.3.1 build zone.
One way to do this is using rsync (make sure you preserve the directory
structure):

```
rsync -va root@<1.6.3-zone-IP>:/root/MY_BITS/* /root/MY_BITS/
```

Once that's complete (still logged into the 13.3.1 build zone with the
LOCAL_BITS_DIR set) you can go to your MG directory and run:

```
(set -o errexit; for TARG in $(tools/targets-13.3.1.sh) platform; do \
    ./configure -t ${TARG} -d joyager -D /stor/donotuse/builds \
        -O /stor/donotuse/builds && make ${TARG}_local_bits_dir; done)
```

This will take quite a while (3-4 hours most likely) but once it's complete,
/root/MY_BITS will contain all the bits required to build the usb headnode
image. To do this, you will want to:

```
git clone git@github.com:joyent/sdc-headnode.git
cd sdc-headnode
make BITS_DIR=/root/MY_BITS usb
```

You can also change 'usb' to 'coal' if you want to build the CoaL image instead.


## Building the platform image

The platform image can be built in a 13.3.1 build zone just like any other
MG target. However there are some additional changes required to these build
zones before you can build platform.

You need to:

 * set fs_allowed="ufs,pcfs,tmpfs"
 * ensure you've got plenty of quota for your zone
 * ensure you've got enough DRAM allocated for your zone

One option for performing all of these at once would be do something like:

 * vmadm update <uuid> fs_allowed="ufs,pcfs,tmpfs" ram=8192 quota=200

from the GZ. This would work fine on hardware but is unlikely to work with CoaL
unless you've bumped the default amount of DRAM for CoaL significantly.

Once you have a properly setup 13.3.1 build zone you can build the platform
with the same command as you'd use for other targets:

```
TARG=platform; ./configure -t ${TARG} -d joyager -D /stor/whatever/builds \
    -O /stor/whatever/builds && make ${TARG}_upload_manta
```

which will build and upload to Manta. Alternatively you can omit the
_upload_manta and just have the platform build to the local bits/ directory.
