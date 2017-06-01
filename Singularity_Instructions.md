# Getting Setup with Singularity

## Pre-requisites (OSX only!)
*Because singularity runs primarily on linux, we need to create a virtual linux environment on OSX in order to build/manipulate singularity containers. Follow these steps first if you're using OSX.*

### Install Homebrew package manager

Homebrew is a package manager for OSX similar to apt-get or yum on linux. It allows you to download and install different software (e.g. wget, or curl) and allows you to build your own packages. Just copy and run the command below:
```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

### Use Homebrew to install Vagrant
Vagrant is a virtual development environment that can be used to create virtual-machines (kind of similar to Virtualbox, but much more powerful). It can be used to install and run another operating system on your computer that's completely independent from your host OS. First we're going to install vagrant via Homebrew.
```
brew cask install Virtualbox
brew cask install vagrant
brew cask install vagrant-manager
```

### Use Vagrant to create a virtual machine
Now that we have vagrant installed, we can use it to make a brand new linux- based virtual machine, **within** which singularity will be installed. It's from inside this vm that we're going to do all future singularity container creation, modification etc. 

First let's create a folder that our virtual machine will live in.
```
mkdir singularity-vm
cd singularity-vm
```
Now lets download a *vagrantfile* for a prebuilt Ubuntu system that already has singularity installed. This command will take a minute to download the necessary files.
```
vagrant init singularityware/singularity-2.3
```
Finally we can start up virtual machine and move into it.
```
vagrant up
vagrant ssh
```
Whenver you're done using a vagrant vm just use ctrl+c to exit the machine and type `vagrant halt` to shut it down.

## Making our first Singularity image
