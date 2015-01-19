========================================================
Creation of Sahara-Compatible HDP 2.0 Images on RHEL 6.6
========================================================

RHEL-OSP's support of Openstack Sahara allows streamlined, reproducible
provisioning of popular data processing clusters including the supported
Hortonworks Data Platform. However, before you can use Sahara to manage your
data processing, you will need to build an image specific to your OS and data
processing engine of choice. The `Sahara Image Elements`_ project is the
maintained Sahara community tool for creation of these images, and it is
built on an Openstack community tool called `Diskimage Builder`_. If you want
to build RHEL Hadoop images for use in your RHEL-OSP Sahara instance, this
document will serve as a guide to use these tools.

.. _`Sahara Image Elements`: https://github.com/openstack/sahara-image-elements
.. _`Diskimage Builder`: https://github.com/openstack/diskimage-builder

As a convenience feature, we are currently in the process of verifying a
packaged image creation flow for RHEL-7-OSP-6. In the meantime, however, this
document and script will provide steps to build a Hortonworks Data Platform
image suitable for running in Sahara, on RHEL 6. While not the very simplest
possible flow, it will keep you as close as possible to the maintained
upstream process for creating HDP images for Sahara, and guide you through
several extra steps to help you use the most supported code available.

TL;DR
=====

You're busy. If you want to read as little as possible, just do this.

* Download the rhel6-hdp20-sahara-image script from `my Github`_.
* Download a RHEL 6.6 qcow2 image from https://openstack.redhat.com/Image_resources.
* Copy both of the above into a RHEL 6.6 server (ideally a new VM, so that you
  don't install any unneeded software on a production server or expose any
  important passwords).
* Log into that server as root, and go to the directory containing those two
  files.
* Run the following commands (substituting your own credentials):

::

    export DIB_RHSM_USER=egafford@redhat.com
    export DIB_RHSM_PASSWORD=im_too_busy_for_my_password
    ./rhel6-hdp20-sahara-image

.. _`my Github`: https://github.com/egafford/rhel6-hdp20-sahara-image

The rest of this document will describe more of what's going on in more detail;
if interested, read on!

Prerequisites
=============

Login as Root
-------------

First, to use this flow, you must be logged in as the root user.

Use a Registered RHEL 6 Build Host
----------------------------------

You'll be building an image from a specific machine, which we'll refer to as
your build host. We'll refer to the VM image that you're building as the
guest, or guest image.

This process has been primarily developed and tested using a RHEL 6 build host
(which may be a VM or physical server.) To follow the path most tested, we
suggest using a RHEL 6 machine, registered to Red Hat Subscription Management.
If you only need the RHEL 6 machine to build this image, of course, feel free
to unsubscribe at the end of the process.

This process *will not* work on RHEL 7 at this time; we are addressing an
incompatibility in that flow, and will release a packaged, streamlined version
of this process when that work is complete.

Step 1: Download a RHEL 6 guest image
=====================================

At present, Hortonworks is supported on RHEL 6.6. To download an appropriate
base image, go here and enter your Red Hat credentials:

https://openstack.redhat.com/Image_resources

Select the RHEL 6.6 image, and place the qcow2 file that you download on your
build host, in the directory from which you intend to run the image creation
script. Then register that file as the base image on which to install HDP:

::

    export ROOT_DIR=`pwd`
    ls ./rhel-guest-image-6.6*.qcow2
    if [ $? != 0 ]; then
        echo "Please download a rhel 6.6 guest qcow2 image to the current working directory."
        exit 1
    fi
    export BASE_IMAGE_FILE=`ls rhel-guest-image-6.6*.qcow2 | tail -n 1`
    export DIB_CLOUD_IMAGES=file://$ROOT_DIR

You'll note that you can also find CentOS and Fedora images; if you prefer to
run your cluster on these operating systems, you may certainly do so! While
this guide does not provide those steps, they are well documented in the
sahara-image-elements repository (linked above.)

Step 2: Set Up Your RHEL Registration
=====================================

Step 2.1: Configure Your Red Hat Login
--------------------------------------

As your RHEL 6 image is installing HDP 2.0, it will need to install a great
deal of software from Red Hat packages. As such, you'll need to provide
credentials to register your guest image (the image will be unregistered
at the end of the flow.) The following code checks to ensure that your
username and password are set:

::

    if [ -z "$DIB_RHSM_USER" ]; then
        echo "Please set env variable DIB_RHSM_USER."
        exit 1
    fi
    if [ -z "$DIB_RHSM_PASSWORD" ]; then
        echo "Please set env variable DIB_RHSM_PASSWORD."
        exit 1
    fi

To pass these checks, run something like this (substituting your information):

::

    export DIB_RHSM_USER=egafford@redhat.com
    export DIB_RHSM_PASSWORD=so_very_secret

Step 2.2: Configure Your Guest's Subscription
---------------------------------------------

The script then sets a few other relevant variables:

::

    export DIB_RHSM_REPOS="rhel-6-server-rpms rhel-6-server-optional-rpms rhel-6-server-openstack-5.0-rpms"
    export DIB_REG_TYPE=portal
    export REG_METHOD=disable
    export REG_HALT_UNREGISTER=true

Note that the REG_METHOD and REG_HALT_UNREGISTER variables are a bit
counterintuitive. They are necessary to tell Diskimage Builder to skip
registration code specific to RHEL 7. Pay them no mind.

Step 3: Install Prerequisites To Your Build Host
================================================

Before we download Diskimage Builder and Sahara Image Elements, we need to
update our build host and install a few necessary virtualization packages. We
also need to download and install the package ``python-argparse`` from EPEL
(Extra Packages for Enterprise Linux). We need to use a slightly newer version
of Diskimage Builder than is packaged for RHEL 6, so EPEL is our only source
for this one dependency. This package is only needed for the image creation
process, not on the image itself, so this unsupported code won't find its way
onto your final image.

::

    yum update -y
    yum install -y qemu-kvm qemu-img kpartx
    cd $ROOT_DIR
    wget http://download.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
    rpm -ivh epel-release-6-8.noarch.rpm
    yum install -y python-argparse
    yum remove -y epel-release

Step 4: Install Diskimage-Builder and Sahara Image Elements
===========================================================

Step 4.1: Install From Git
--------------------------

Next we'll install Diskimage Builder and Sahara Image Elements from upstream
source. While Red Hat does package Diskimage Builder for RHEL 6 (in RHOS 5,)
we need the Juno version specifically (the version from which RHOS 6 is
built,) so we need to install from Git.

::

    yum install -y git
    if [ ! -d $ROOT_DIR/sahara-image-elements ]; then
        git clone https://github.com/openstack/sahara-image-elements.git
    fi
    if [ ! -d $ROOT_DIR/diskimage-builder ]; then
        git clone https://github.com/openstack/diskimage-builder.git
    fi
    cd $ROOT_DIR/sahara-image-elements
    export SAHARA_ELEMENTS_COMMIT_ID=2014.2
    git checkout $SAHARA_ELEMENTS_COMMIT_ID
    cd $ROOT_DIR/diskimage-builder
    export DIB_COMMIT_ID=0.1.29
    git checkout $DIB_COMMIT_ID
    cd $ROOT_DIR

Step 4.2: Remove EPEL Installation On the Guest Image
-----------------------------------------------------

One last line is necessary here: the upstream Hortonworks Data Platform code
uses some packages from EPEL (specifically packages for Nagios, a monitoring
tool) on the guest image. Because we're registering our guest image for RHOS,
however, we can instead install Nagios from supported packages. As such, it's
best not to install EPEL on the guest at all. We can remove the offending
installation as follows:

::

    sed -i '/epel-release/d' $ROOT_DIR/sahara-image-elements/elements/hadoop-hdp/install.d/30-init-hdp-install

Step 4.3: Configure the Diskimage-Builder Elements Path
-------------------------------------------------------

Diskimage Builder will add a number of 'elements' to your base image. Sahara
Image Elements provides several additional elements, specific to Sahara, but
by default, Diskimage Builder doesn't know that these exist. As such, you'll
want to create the following variables.

::

    export BASE_ELEMENTS_PATH="$ROOT_DIR/diskimage-builder/elements"
    export SAHARA_ELEMENTS_PATH="$ROOT_DIR/sahara-image-elements/elements"
    export ELEMENTS_PATH=$BASE_ELEMENTS_PATH:$SAHARA_ELEMENTS_PATH

Step 5: Create Your HDP Image
=============================

Step 5.1: Configure HDP Settings
--------------------------------

A few last configuration settings are necessary to configure the HDP element
itself. Most notable is DIB_HDP_VERSION, which will install HDP 2.0 (rather
than the older 1.3).

::

    export DIB_IMAGE_SIZE=10
    export DIB_HDP_VERSION=2.0
    export JAVA_TARGET_LOCATION=/opt
    export JAVA_DOWNLOAD_URL=https://s3.amazonaws.com/public-repo-1.hortonworks.com/ARTIFACTS/jdk-6u31-linux-x64.bin

Step 5.2: Run Diskimage Builder
-------------------------------

Having configured all of the above, we need only run Diskimage Builder. The
following command will open the base RHEL 6.6 image you downloaded above,
install HDP on it, and package it into a new, Openstack-ready VM:

::

    $ROOT_DIR/diskimage-builder/bin/disk-image-create "rhel hadoop-hdp redhat-lsb vm" -n -o sahara-rhel-6.6-hdp-2.0

Note: during the installation of Java from Hortonworks' repository, the
process opens a documentation page for Java. Just press Enter until the
process continues (*not* Ctrl-C).

At the end of this flow, the script will emit a file called
``sahara-rhel-6.6-hdp-2.0.qcow2``. This is the image you'll use in Sahara.

Step 6: Register the Image with Sahara
======================================

First, register the image with Glance, using either the UI or the API. In the
UI, you can find this in your project's interface at Compute/Images.

Second, register this image with Sahara. To use the UI, go to the interface at
Project/Data Processing/Images.

* Select Register Image.
* Select the Glance image you just created.
* Name the image something useful, specific, and memorable (rhel-66-hdp-20 or
  somesuch).
* Add tags for your plugin (hdp and 2.0.6). NOTE: you must click the "Add
  Plugin Tags" button next to this interface for the tags to be added; without
  this step, Sahara will not find your image as a candidate for a new cluster.
* Register your image.

You are now ready to begin using Sahara: creating cluster topologies, spinning
up clusters, and running elastic data processing jobs. We hope this guide was
helpful in getting you started. Happy stacking!
