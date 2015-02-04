cloudify-nodecellar-docker-example
==================================

Sample application running inside docker containers

## Single-host Walk Through

Nodecellar is a nodejs frontend that uses a mongo database as its backend. In this example each software component of the application is run inside of its own Docker contianer.

Start out by installing Cloudify on your local machine. You can install Cloudify on Windows, Mac, or Linux. Visit the instructions [here.](http://getcloudify.org/guide/{{page.cloudify_version}}/installation-cli.html) In these instructions, we just install from pip:

`pip install cloudify`

Now we need to pull the git repository:

`wget https://github.com/cloudify-cosmo/cloudify-nodecellar-docker-example/archive/3.1.zip`

The blueprint for the example is in the blueprint directory: docker-singlehost-blueprint.yaml. There is also an Openstack example for you to try there as well.

First notice the files that we import:

    imports:
      - http://www.getcloudify.org/spec/cloudify/{{page.cloudify_version}}/types.yaml
      - https://raw.githubusercontent.com/cloudify-cosmo/cloudify-docker-plugin/{{page.plugin_version}}/plugin.yaml


The nodes in the blueprint derive their base properties from these two files. They should match in the minor version. (3.1 and 1.1, or 3.0 and 1.0.)

Notice the first node_template: host. This describes the host machine that the Docker containers will run on. In this case it will be your local computer. However, this could be adjusted to be a vagrant box or something else. 

If you decide to run the containers somewhere else, you will need to adjust the values in the inputs section or use an external inputs file.

    host:
      type: cloudify.nodes.Compute
      properties:
        install_agent: { get_input: install_agent }
        ip: { get_input: host_ip }
        cloudify_agent:
          user: { get_input: agent_user }
          key: { get_input: agent_private_key_path }


Now let's look at the Nodecellar Container.

In this example, we are going to have a container that runs the nodejs nodecellar application. The Docker plugin will pull the uric/nodecellar container from Docker Hub.

    nodecellar_container:
      type: cloudify.docker.Container
      properties:
        name: nodecellar
        image:
          repository: uric/nodecellar
        ports:
          8080: 8080
        params:
          stdin_open: true
          tty: true
          command: nodejs server.js
          environment:
            NODECELLAR_PORT: 8080
            MONGO_PORT: 27017


The plugin configures the container to expose port 8080, with a pseudo TTY, and two environment variables NODECELLAR_PORT, and MONGO_PORT.

Notice that port 8080 is mapped to port 8080. You could change this to map port 80 to 8080 or any other combination.


Next is the MongoDB container. The plugin will pull the dockerfile/mongodb image from Dockerhub, and create a container that exposes ports 27017 and 28017.

    mongod_container:
      type: cloudify.docker.Container
      properties:
        name: mongod
        image:
          repository: dockerfile/mongodb
        ports:
          27017: 27017
          28017: 28017
        params:
          stdin_open: true
          tty: true
          command: mongod --rest --httpinterface --smallfiles


The plugin then starts the container with a pseudo TTY and runs the command `mondod --rest --httpinterface --smallfiles`. Again the ports 27017 and 28017 are mapped to themselves, but you could change them to different mappings if you configured MongoDB to other ports.

## Execute the Operations

Now, let's get Cloudify setup so you can run the plugin.

Start by installing the plugins:

`cfy local install-plugins -p docker-singlehost-blueprint.yaml`

Now initialize the environment:

`cfy local init -p docker-singlehost-blueprint.yaml`

Now you are ready to run the blueprint:

`cfy local execute -w install`

This might take a while to execute. It needs to install download the Docker images and then it will create the containers.

When it is finished, you can open a browser to 127.0.0.1 and you will see the Nodecellar application.

# Running the example inside of a manager

## Vagrant

First download the Cloudify 3.1 Vagrantfile:

`wget http://gigaspaces-repository-eu.s3.amazonaws.com/org/cloudify3/3.1.0/ga-RELEASE/Vagrantfile`

Then add the Vagrant box:

`vagrant box add http://gigaspaces-repository-eu.s3.amazonaws.com/org/cloudify3/3.1.0/ga-RELEASE/cloudify-virtualbox_3.1.0-ga-b85.box --name=cloudify-box`

When the download is finished, you can "up" the box:

`vagrant up`

Now, you can ssh into the box:

`vagrant ssh`

You need to install Docker, because the plugin's Docker installation script only works with Ubuntu Precise:

`curl -sSL https://get.docker.com/ubuntu/ | sudo sh`

`sudo gpasswd -a ${USER} docker`

Now logout and log back in:

`exit`

`vagrant ssh`

Change into the blueprints directory and download this example:

`cd blueprints`


`git clone https://github.com/cloudify-cosmo/cloudify-nodecellar-docker-example.git`

And checkout the version:

`git checkout tags/3.1`

There are some minor changes to use the single host example in vagrant. So in that case you will use the vagrant_inputs.json file.

`cfy blueprints upload -b nodecellar -p blueprint/docker-singlehost-blueprint.yaml`

Create a deployment:

`cfy deployments create -b nodecellar -i cfy-vagrant-inputs.json -d nodecellar`

And execute the install workflow:

`cfy executions start -w install`

After the deployment runs, you'll be able to go to the following URL and visit the Nodecellar application: http://11.0.0.7:8080/.

## Openstack

This is an example running the Openstack blueprint.

The Openstack is slightly different than what we have described:

* Creates a virtual machine in Openstack.
* Installs Docker on the virtual machine.
* Pulls the respective images and runs the containers on the virtual machine.
* Creates two security groups: Nodecellar, and Mongo, and attaches them to the manager.
* There is a cloudify.relationships.connected_to relationship between the Nodecellar and Mongo security groups, which might strike you as odd. This is a temporary work-around for an Openstack issue where simultaneously connecting security groups to a server might fail without an error. This work-around creates a dependency between these security groups to make sure that one is attached first, and then the other.

Conversely, the singlehost:

* Runs locally.
* Does not install Docker.
* Pulls the respective images and runs the containers locally.
* There are no additional resources like the security group.

When it comes to deploying from a manager, the process is essentially the same between blueprints:

Start by initializing the environment.

`cfy init`

Now tell Cloudify, the URL of your manager:

`cfy use -t your-manager-ip`

Upload the blueprint:

`cfy blueprints upload -p docker-openstack-blueprint.yaml -b nodecellar-docker`

Create a deployment:

`cfy deployments create -b nodecellar-docker -d deploy-blueprint`

Now you can execute it:

`cfy executions start -w install -d deploy-blueprint`

Again this may take a few moments to run, but when you are finished, you can direct a browser to the virtual machine created by the execution and you will see Nodecellar there.
