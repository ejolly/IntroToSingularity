# Getting Setup with Singularity
*This is a guide to getting started with [Singularity containers](http://singularity.lbl.gov/) in conjunction with Dartmouth College's [Discovery HPC](http://techdoc.dartmouth.edu/discovery/).  
Questions can be addressed to eshin.jolly.gr@dartmouth.edu or mvdoc.gr@dartmouth.edu.  
We're not experts but we're happy to try to help!*

#### [I. Pre-requisites (OSX only)](#prereqs)
#### [II. Creating a Singularity container](#creation)
#### [III. Basic container usage](#basicusage)
#### [IV. Using a container on Discovery](#discovery)
#### [V. Updating an existing container](#updating)
#### [VI. Sharing containers](#sharing)
#### [VII. Extra resources](#resources)


## <a name="prereqs"></a> Pre-requisites (OSX only!)
*Because singularity runs primarily on linux, we need to create a virtual linux environment on OSX in order to build/manipulate singularity containers. Follow this step first if you're using OSX.*

#### Install Homebrew package manager

Homebrew is a package manager for OSX similar to apt-get or yum on linux. It allows you to download and install different software (e.g. wget, or curl) and allows you to build your own packages. Just copy and run the command below:
```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

#### Use Homebrew to install Vagrant
Vagrant is a virtual development environment that can be used to create virtual-machines (kind of similar to Virtualbox, but much more powerful). It can be used to install and run another operating system on your computer that's completely independent from your host OS. First we're going to install vagrant via Homebrew.
```
brew cask install Virtualbox
brew cask install vagrant
brew cask install vagrant-manager
```

#### Use Vagrant to create a virtual machine
Now that we have vagrant installed, we can use it to make a brand new linux- based virtual machine, **within** which singularity will be installed. It's from inside this vm that we're going to do all future singularity container creation, modification etc.

First let's create a folder that our virtual machine will live in.
```
mkdir singularity-vm
cd singularity-vm
```
Now lets download a *vagrantfile* for a prebuilt Ubuntu system that already has singularity installed.
```
vagrant init singularityware/singularity-2.3
```
Finally we can start up virtual machine and move into it.
```  
#If this is the first time you're building the vm the vagrant up command might take a minute or so to complete
vagrant up
vagrant ssh
```
Whenver you're done using a vagrant vm just use `ctrl+c` to exit the machine and type `vagrant halt` to shut it down.

## <a name="creation"></a>Creating a Singularity container
Let's begin by creating a new folder within our vm for our brand new container (this isn't strictly necessary but nice to keep different containers organized):
```
mkdir miniconda
cd miniconda
```
The first thing we need to do in order to create a singularity container is make a singularity *definition* file. This is just an instruction set that singularity will use to create a container. Think of this definition file as a recipe, and the container as the final product. Within this recipe, specify everything you need to in order create your custom analysis environment. Sharing this definition file with others will enable them to identically reproduce the steps it took to create your container.  

To get you started here's an example definition file that we're going to use for this demo. This is a simple neurodebian flavored container with miniconda installed along with numpy and scipy.  
Let's save this to a file called `miniconda.def`

```  
# Singularity definition example with miniconda
# Matteo Visconti di Oleggio Castello; Eshin Jolly
# mvdoc.gr@dartmouth.edu; eshin.jolly.gr@dartmouth.edu
# May 2017

bootstrap: docker
from: neurodebian:jessie

# this command assumes singularity 2.3
%environment
    PATH="/usr/local/anaconda/bin:$PATH"
%post
    # install debian packages
    apt-get update
    apt-get install -y eatmydata
    eatmydata apt-get install -y wget bzip2 \
      ca-certificates libglib2.0-0 libxext6 libsm6 libxrender1 \
      git git-annex
    apt-get clean

    # install anaconda
    if [ ! -d /usr/local/anaconda ]; then
         wget https://repo.continuum.io/miniconda/Miniconda2-4.3.14-Linux-x86_64.sh \
            -O ~/anaconda.sh && \
         bash ~/anaconda.sh -b -p /usr/local/anaconda && \
         rm ~/anaconda.sh
    fi
    # set anaconda path
    export PATH="/usr/local/anaconda/bin:$PATH"

    # install the bare minimum
    conda install\
      numpy scipy
    conda clean --tarballs

    # make /data and /scripts so we can mount it to access external resources
    if [ ! -d /data ]; then mkdir /data; fi
    if [ ! -d /scripts ]; then mkdir /scripts; fi

%runscript
    echo "Now inside Singularity container woah..."
    exec /bin/bash

```

Now lets use our vagrant vm and create a blank singularity image allocating 4gb of disk space within our container. You may need to adjust this depending on how much software you plan to install. By default the vagrant vm will share `/vagrant` with your host OS so lets perform our operation in there within the container folder we created earlier.
```
vagrant up
vagrant ssh
cd /vagrant/miniconda
sudo singularity create --size 4096 miniconda.img
```
Finally lets actually build the container using our definition file. The amount of time it takes to build a container will vary wildly depending on what different programs you install and how fast your internet connection is.
```
sudo singularity bootstrap miniconda.img miniconda.def
```

## <a name="basicusage"></a>Basic container usage  

If all went well we should be able to issue a python command to the python version installed *within* our container like so:
```
singularity exec miniconda.img python -c 'print "Hello from Singularity!"'
```
We can also open up our container and work *inside* it interactively:
```
singularity run miniconda.img
conda list
```
Press `ctrl+d` to exit the container.

Most commonly you'll use one of three commands with a container:  
`singularity exec` to run a specific command/file/script using the container  
`singularity run` to move into a container and use it interactively; what   gets run by this command is dictated by your singularity *definition* file  
`singularity shell` similar to above, but specifically open up a shell within the container  

A few other useful flags include:
`-B` mount an external folder to the container  
`-c` don't automatically map /home and /tmp to shared folders with the host OS  

## <a name="discovery"></a>Using a container on Discovery  

In order to use a container on Discovery you have to first upload the generated .img file to your home directory. Since containers can be rather large lets compress this and then uncompress on Discovery (starting with Singularity >=2.3.0 this functionality works through `import` and `export` commands)
```
tar -cvzf miniconda.tar.gz miniconda.img
scp miniconda.tar.gz ejolly@discovery.dartmouth.edu:~
ssh ejolly@discovery.dartmouth.edu
tar -xvzf miniconda.tar.gz
```
Now you can utilize the container by loading the singularity module and utilizing any of the singularity commands above. There is **one catch** however: by default singularity will try to melt together any environment variables defined in your account on discovery with environment variables defined within the container. The rationale behind this is that singularity offers the ability to *seamlessly* blend a custom environment (i.e. your container built with all your goodies) and the functionality of your HPC (i.e. all the goodies that already exist on Discovery). However, often times you want to turn this functionality off and only use environment variables within your container to avoid conflicts (i.e. completely ignore environment variables set on Discovery). Here's how we do that:
```  
module load singularity

env -i `which singularity` run miniconda.img
#Or on singularity >=2.3.0
singularity run -e miniconda.img
```

To make our lives easier we can create a simple bash script that executes a command in our container making sure to call it with all the extra flags we want (e.g. mounting some folders, ignoring environment variables). I personally like to create two scripts one for interactively working with a container and one for using it to execute commands for example with job submission. Here are some examples, you'll need to adapt them to mount the directories you want:  
Let's save the following code into a bash file called: exec_miniconda
```  
#!/bin/bash
env -i `which singularity` exec \
    -B /idata/lchang/Projects:/data \
    -B /ihome/ejolly/scripts/:/scripts \
    miniconda.img "$@"
```
Let's save the following code into a bash file called: interact_miniconda
```  
#!/bin/bash
env -i `which singularity` run \
	-c \
	-B /idata/lchang/Projects/Pinel:/data \
	-B ~/scripts:/scripts \
	miniconda.img
```

Now we issue a command to our container (e.g. when submitting a job) like this:
```
./exec_miniconda python -c 'print "Hello World!"'
```

We can also use our container interactively with. Here let's actually serve a jupyter notebook server from the cluster and interact with it using our local web browser. To do so we need to reconnect to Discovery with port-forwarding.  The demo container here isn't built with a jupyter notebook so this won't work, but we you can use the same command when building your own container
```  
# You should really connect to something other than the head node here!
ssh ejolly@discovery.dartmouth.edu -N -f -L localhost:3129:localhost:9999

./exec_miniconda jupyter notebook --no-browser --port=9999
# On local machine navigate to localhost:3129 in a web browser
```

## <a name="updating"></a>Updating an existing container  

The preferred way to update a container is to modify the definition file and rebuild the image using the steps above. This ensures that any container image is always a product of its definition file and is therefore easy to reproduce.

However, singularity makes it easy to make changes to an existing container as well using the `--writable` flag with the `exec`, `run`, or `shell` commands, e.g.
```
singularity exec --writable miniconda.img apt-get install curl
```
You can also increase the size of an existing container with the `expand` command, e.g.
```  
#Expand a container by 2gb
singularity expand --size 2048 miniconda.img
```

## <a name="sharing"></a>Sharing containers  

One of the nice things about using singularity (and containers in general) is that you can share your analysis environment with others. These are served on [Singularity hub](https://singularity-hub.org). Many prebuilt containers already exist that you easily download and use.

Let's say we want to use this [container](https://singularity-hub.org/containers/105/) prebuilt with tensor flow for GPUs. This is as simple as:
```
singularity pull shub://researchapps/tensorflow:gpu
```
Then you can setup run and execute scripts like above to use it on Discovery.

You can also easily share you custom container on Singularity hub by committing your singularity definition file to github and flipping the switch for that repository on singularity hub.

## <a name="resources"></a>Extra resources
Much of this tutorial is borrowed/integrated from several helpful resources:

#### Quick guides
[Matteo Visconti's blogpost on getting started with singularity](http://mvdoc.me/2017/using-singularity-to-make-analyses-reproducible.html)  
[Jin Cheong's quick guide to using the discovery cluster](http://jinhyuncheong.com/jekyll/update/2016/07/24/How-to-use-the-Discovery-cluster.html)

#### More comprehensive guides
[Singularity Documentation](http://singularity.lbl.gov/quickstart)  
[Discovery Documentation](http://techdoc.dartmouth.edu/discovery/)

#### Sharing is caring
[Singularity hub](https://singularity-hub.org/)  
[Docker hub](https://hub.docker.com/)
