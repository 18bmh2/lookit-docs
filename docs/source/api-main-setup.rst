==================================
Lookit API Local Machine Setup
==================================

Prerequisites
~~~~~~~~~~~~~
Before you can get the API running on your machine, there are a few steps you need to take to ensure everything will work
properly.

Install Docker
--------------
**For Linux Users**

Follow  `these instructions <https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-18-04>`_.
Make sure you select the proper OS.


Install Postgres
----------------

**For Linux Users**

Before you get started, update your system with this command:

``sudo apt-get update``

Make sure you have python3 and pip installed:

``sudo apt install python3``

``sudo apt install python-pip``

Now, begin to install Postgres:

``sudo apt-get install PostgreSQL PostgreSQL-contrib``

Run the following command. It will take inside the Postgres world.

``sudo -u postgres psql postgres``

Every command now should start with postgres=#

In the postgres world, run the following commands:

``#\password postgres``

You should be prompted to enter a new password. Don’t type anything, just hit enter twice. This should clear the password.

postgres=# ``create database lookit``

postgres=# ``grant all privileges on database lookit to postgres``

If at this point you still do not have access to the lookit database, run the following commands:

``sudo vi /etc/postgresql/10/main/pg_hba.conf``

``pg_ctl reload``

A long document with # on the leftmost side of almost every line should open up. Scroll to the bottom. There will be a few lines that don’t start with #. They might be a different color and will start with either local or host. The last word in each of those lines should be trust. If it is not, switch into editing mode ( hit esc then type i and hit enter ) and change them to say trust. Then, save the file ( hit esc and then type :x before hitting enter ). You should now have access.





Install RabbitMQ
----------------

**For Linux Users**

First, run the following command:

``sudo apt install rabbitmq-server``

Now that rabbitmq server is installed, create an admin user with the following commands:

``sudo rabbitmqctl add_user admin password``

``sudo rabbitmqctl set_user_tags admin administrator``

``sudo rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"``

Make sure that the server is up and running:

``sudo systemctl stop rabbitmq-server.service``

``sudo systemctl start rabbitmq-server.service``

``sudo systemctl enable rabbitmq-server.service``

If you are having problems creating a user or getting the server running, try the following commands:

``sudo rabbitmq-plugins enable rabbitmq_management``

``sudo chown -R rabbitmq:rabbitmq /var/lib/rabbitmq/``

``sudo rabbitmqadmin declare queue --vhost=/ name=email``

``sudo rabbitmqadmin declare queue --vhost=/ name=builds``

``sudo rabbitmqadmin list queues``

When you run the last command, you should see something like this: ::

    +--------+----------+
    | name  | messages  |
    +--------+----------+
    | builds|     0     |
    | email |     0	|
    +--------+----------+


Install Ngrok
-------------

**For Linux Users**

To install, run this command:

``sudo snap install ngrok``

To connect to your local host run this command:

``ngrok http “[https://localhost:8000](https://localhost:8000)"``



Install SSL Certificates
------------------------



Setup
~~~~~


How do these things work?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Docker
------


Docker is used as an alternative to virtual machines. Docker Makefiles, or *Dockerfiles*, are recipes containing all
dependencies and the library needed to build an *image*. An *image* is a template that is built and stored and acts as
a model for *containers*, analogous to the class-object relationship in object-oriented programming. It contains the
application plus the necessary libraries and binaries needed to build a *container*. Since they are templates, *images*
are what can be shared when exchanging containerized applications. On the other hand, containers are an ephemeral running
of a process, which makes it impossible to share them. Instances of the class, or objects, are to Class as a *container*
is to an *image*. A *container* is a running instance of an *image*. [*]_

Docker is software that allows you to create isolated, independent environments, much like a virtual machine. To
understand how Docker accomplishes these feats, you must first understand both the PPID/PID hierarchy of the Linux
kernel and union file systems.

When using the Linux OS, every program running on your machine as a process ID (PID) and a parent process ID (PPID).
Some programs (parent programs) have the ability to launch other programs (child programs). The PID of the original
program becomes the PPID of the child process. This system forms a tree of processes branching off of one another.
These ID numbers correspond to which port the programs are running on.

The root node of this "process tree" is called ``systemd``, a.k.a. system daemon. It has PID 1 and PPID 0. In older
distributions of Linux, the ``init`` process was used. [#]_ The job of this process is to launch other processes and adopt
any orphaned processes (child processes whose parent processes have been killed). All of these programs can interact
with each other through shared memory and message passing method.

Shared memory is easily understood via the Producer, Consumer analogy. Imagine you have two processes, a producer
process and a consumer process. When the producer process creates a good, it will put it in a store where the consumer
process can find and consume it. This store is shared memory.

The message passing method utilized communication links to connect two processes and allow them to send messages to each
other. After a link is established, processes can send messages that contain a header and a body. The header contains
the metadata, such as the type of message, recipient, length, etc.

These communication methods allow for processes to interact with each other, and, as you can imagine, this creates a
problem when it comes to isolation.

Programs can interact with each other when they're in the same *namespace*, which I refer to as a ``systemd`` tree. [#]_ In
other words, all the programs that stem from the same ``systemd`` command can interact with each other, but if you were
to have a second ``systemd`` tree, its programs could be isolated from those of the first. This is where Docker comes in.
Docker creates a new namespace, a.k.a. a *userspace*. When the Docker process is run, it is launched from the real
``systemd`` process and given a normal PID. A new feature released by Linux in 2008, however, allows Docker to have more
than one PID. With this new feature, *userspaces* come with a table that can map a relative PID seen by your container
in said *userspace* to the actual PID seen by your machine. This allows Docker to take on PID 1 in your container.

Now, all programs that stem from the Docker branch will see Docker in the same way they see the ``systemd`` process. It is
the root node, and they will not leave the branch, effectively cutting off interaction between those processes and the
ones on the rest of the real ``systemd`` tree.

Though this is a step closer to isolation, it is not quite there yet. Even though processes can't interact with the main
branch, they can still interact with the main filesystem. To combat this, Docker makes use of union filesystems. [#]_ A union
filesystem uses union mounts to merge two directories into one. This means that if you have two directories that contain
different files, a union mount will create a new directory that contains all of the files from both. If both directories
contain a file of the same, the mount usually has a system in place for which one it will choose.

One big thing that makes this file system important for Docker is its deletion process. When you delete a file in a
union filesystem, it does not actually delete it, rather it adds an extra layer and masks it. This masking process
allows Docker to unionize your machine's filesystem and your Docker filesystem, masking all files that are specific to
your machine. The necessary directories for Linux set up, such as the ``/etc``, ``/var``, ``/``, ``usr``, and ``home`` directories
are still intact, however extra, added files from your machine will be masked from Docker. In addition, when you write
files to this union filesystem, they will not be written to your machine's file system, effectively isolating your
machine from your containers.

Another big difference between Docker and Virtual Machines is the Hypervisor. [#]_  When using a VM, a hypervisor is
necessary to supervise, schedule, and control interactions between the host OS and the Guest OS. It acts as a layer of
security between your machine and the virtual one so that yours is not damaged or messed with. Docker eliminates the
need for the hypervisor because there is no longer a Guest OS. The Docker Engine is software downloaded directly onto
your machine, and the containers run on the engine. Using Docker eliminates extra steps needed for the VM, as it doesn't
have to virtualize an entire computer. This makes Docker faster, more efficient, and less redundant than VMs.

.. image:: https://cdnssinc-prod.softserveinc.com/img/blog/containers-security-virtual-machines.PNG

image credits: `docker.com <https.docker.com>`_

For a more in-depth explanation of Docker and how it works, consder looking at `this series of articles.
<https://www.nschoe.com/articles/2016-05-26-Docker-Taming-the-Beast-Part-1.html>`_


Postgres
--------

PostgreSQL is a general purpose and object-relational database management system. It allows you to store and query large
amounts of data without having to worry about all of the complicated processes going on under the hood. [*]_  PostgreSQL
optimizes data querying for you, making your application faster. All information and metadata is stored in it.




RabbitMQ
---------

RabbitMQ is a message broker. When messages are sent online, they go from the producer to a queue and then to the
consumer. RabbitMQ is that queue. Instead of having to perform all tasks involving sending messages, including generating
PDFs, locating the recipient, etc., the message producer only has to upload their message and instructions to the queue
and RabbitMQ will take care of the rest. Using this service makes messaging through Lookit easier and more efficient, as
it is able to re-queue messages, it is faster and more reliable, and it is scalable for when there are a lot of messages.

`source <https://www.cloudamqp.com/blog/2015-05-18-part1-rabbitmq-for-beginners-what-is-rabbitmq.html>`_


Ngrok
-----

Ngrok is used in the development process to act as a tunnel into your PC. It is not secure to allow access into your PC
or local address through a public channel, as this can open you up to malicious attacks. Ngrok allows you to securely
provide acces to your local address through something public, like the internet. When Ngrok is running on your machine,
it gets the port of the network service, which is usually a web server, and then connects to the Ngrok cloud service.
This cloud service takes in traffic and requests from a public address and then relays that traffic directly to the
Ngrok process running on your machine, which then passes it along to the local address you specify. By doing this, you
can minimize the interaction between outside traffic and your personal machine.

When trying to stream videos on the development stage, AWS will need an address to send the video to. Ngrok will create
a dummy link for this purpose and then send the video from this dummy address to your PC.




External Resources
~~~~~~~~~~~~~~~~~~~

Google Cloud
-------------

The Cloud service is where all the studies are stored.


Amazon Web Services
--------------------

This is where the consent videos are stored??

Celery
-------

This is what runs the long term tasks

Authenticator
---------------

Allows you to log into your account securely

Lookit Ember Frameplayer
------------------------

Consent manager videos

ADDPIPE
-------

ADDPIPE is used to record the video and audio. It connects to the hardware of your computer and films for you. It also
converts recorded files to ,mp4. https://addpipe.com/about

How Do These Programs Work Together?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The following diagrams illustrate how different parts of the API interact with each other.

.. image:: _static/img/RabbitMQ.png

Every time the user makes a request, it is sent through microWSGI. If it is a request that will take more than a few
seconds, it is sent off to RabbitMQ which will delegate the tasks one by one to celery. If it is short enough, HTTP will
handle the request.

.. image:: _static/img/celery.png

Celery is used to build and relay tasks and make the API more efficient. Lengthy requests are sent through RabbitMQ to
celery, which will complete them on the side. The tasks sent to celery are ones that would ruin the user experience if
they backlogged the HTTPs request cycle. Programs like celery are used to keep the request cycle short.


.. image:: _static/img/docker.png

When you want to build a study, celery sends that request to Docker, which then sends the study static files back to celery.
After building the study, celery sends deployable static files to Google Cloud.





.. image:: _static/img/use-case.png

This is a diagram of all interactions possible with the Lookit API. On the rightmost side are all external resources being
used/



Footnotes
----------


.. [#] If you’re interested in learning about the difference between ``init`` and ``systemd`` as well as the reasoning behind the switch, check
       out this `link <https://www.tecmint.com/systemd-replaces-init-in-linux/](https://www.tecmint.com/systemd-replaces-init-in-linux/>`_.
.. [#] The Linux kernal has many built in namespaces that are responisble for different things. If you are interested in
        learning more about this topic, check out this article on `namespaces <https://medium.com/@teddyking/linux-namespaces-850489d3ccf>`_
.. [#] The union filesystem utilizes set theory. For a more in depth explaination of how they work and the math behind them,
       check out this article on `union filesystems <https://medium.com/@paccattam/drooling-over-docker-2-understanding-union-file-systems-2e9bf204177c>`_
.. [#] Hypervisors are essential to the functionality of VMs. If you want to know more about them, check out this link on
       `hypervisors <https://www.networkworld.com/article/3243262/what-is-a-hypervisor.html>`_


Endnotes
---------
.. [*] Docker has many other moving parts behind the scenes. An example of a part is volumes. Volumes serve as a storage
       space for your containors. For more in depth information about volumes, check out this `link <https://blog.container-solutions.com/understanding-volumes-docker>`_
       In addition, this `series of articles <https://www.nschoe.com/articles/2016-05-26-Docker-Taming-the-Beast-Part-1.html>`_
       covers a lot of Docker topics not mentioned in this documentation.
.. [*] `add foot/endnote on what postgres is doing behind the scenes <https://medium.com/@divya.n/how-postgres-works-733bc5cf61a>`_