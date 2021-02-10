Installation
============

Zenodo depends on PostgreSQL, Elasticsearch 2.x, Redis and RabbitMQ.
This is my steps to install and run a local development instance of zenodo on my Ubuntu 20.04 development environment.

----------------------

Docker installation

.. code-block:: console

    $ sudo curl -L "https://github.com/docker/compose/releases/download/1.28.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    $ sudo chmod +x /usr/local/bin/docker-compose
    $ sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
    $ docker-compose --version



--------------------

Python installation

See `this guide <https://phoenixnap.com/kb/how-to-install-python-3-ubuntu/>`_ for details.


in /bashrc
sudo gedit ~/.bashrc

Insert these lines:

.. code-block:: console

    #virtualenvwrapper
    export WORKON_HOME=~/Envs
    #export WORKON_HOME=$HOME/.virtualenvs
    export VIRTUALENVWRAPPER_PYTHON=/usr/bin/python3.8
    export VIRTUALENVWRAPPER_VIRTUALENV=/usr/local/bin/virtualenv
    source /home/ajlite/.local/bin/virtualenvwrapper.sh
    #source /usr/local/bin/virtualenvwrapper.sh


Important!!!!

.. code-block:: console

    $ sudo chown -R uname ~/Envs

-------------------

Zenodo Development Installation

For the development setup we will reuse the Zenodo docker image from
previous section to run only essential Zenodo services, and run the
application code and the Celery worker outside docker - you will want to
have easy access to the code and the virtual environment in which it will be
installed.

.. code-block:: console

    $ cd ~/src/
    $ git clone https://github.com/osct/zenodo.git
    $ cd ~/src/zenodo
    $ git checkout master
    ////see the following notes before the following step
    //nvm use v10.19.0
    //npm install npm@latest -g
    $ docker-compose up -d
  
 
.. note::

    Before doing the last step .. Change some lines in the Dockerfile which is in ~/src/zenodo directory:

.. code-block:: console
    
    # Node.js 
    && curl -sL https://deb.nodesource.com/setup_10.x | bash - \
    && apt-get -qy install --fix-missing --no-install-recommends \
      nodejs \
    && npm install -g npm@4 \

.. note::

     Before doing the last step .. Change some lines in the reqiurements.txt file which is in ~/src/zenodo directory:
     
.. code-block:: 

    MarkupSafe==1.1.0
    maxminddb==1.5.2
    MarkupSafe==1.1.0

---------------------------------------

Keep the docker-compose session above alive and in a separate shell, create a
new Python virtual environment using virtualenvwrapper
(`virtualenvwrapper <https://virtualenvwrapper.readthedocs.io/en/latest/>`_),
in which we will install Zenodo code and its dependencies:

.. note::

    Zenodo works on both on Python 2.7 and 3.5+. However in case you need to
    use the XRootD storage interface, you will need Python 2.7 as the
    underlying libraries don't support Python 3.5+ yet.
    I will use python 2.7 // for infn use 3.5
    
    
.. code-block:: console

    $ mkvirtualenv -p python2.7 ajlite
    (ajlite)$


Next, change these versions in /src/zenodo/requirements.txt
.. code-block:: console
        psycopg2-binary==2.8.6
        ##scipy==1.4.1    //if python3.8
        dulwich==0.20.2


Next, install Zenodo and code the dependencies:

.. code-block:: console

    (ajlite)$ cd ~/src/zenodo
    (ajlite)$ sudo apt-get install build-essential python-dev
    /////the same as version of python of the environment
    (ajlite)$ sudo apt install libffi-dev
    (ajlite)$ pip install cffi==1.12.3
    (ajlite)$ pip install pyOpenSSL
    (ajlite)$ sudo apt install libssl-dev
    (ajlite)$ pip install --default-timeout=100000 -r requirements.txt --src ~/src/ --pre --upgrade
    (ajlite)$ pip install -e .[all,postgresql,elasticsearch2]


Media assets
~~~~~~~~~~~~

Next, we need to build the assets for the Zenodo application.

To compile Zenodo assets we will need to install:

* NodeJS **7.4** and NPM **4.0.5**

* Asset-building dependencies: SASS **3.8.0**, CleanCSS **3.4.19**, UglifyJS **2.7.3** and RequireJS **2.2.0**

Open new terminal window:

.. code-block:: console

   $ sudo apt install curl
   $ curl https://raw.githubusercontent.com/creationix/nvm/master/install.sh | bash
   $ source ~/.profile  
   $ nvm install v7.4

Once NVM is installed, set it to use NodeJS in version 7.4:

.. code-block:: console

   (zenodo)$ nvm use 7.4
   Now using node v7.4.0 (npm v4.0.5)

Or

.. code-block:: console

   (zenodo)$ sudo ln -s "$(which node)" /usr/bin/node
   (zenodo)$ sudo ln -s "$(which npm)" /usr/bin/npm
   (zenodo)$ sudo ./scripts/setup-npm.sh


As before, install the npm requirements, this time without ``sudo``:

.. code-block:: console

   (zenodo)$ ./scripts/setup-npm.sh

the packages will be installed in your local user's NVM environment.

After you've installed the NPM packages system-wide or with NVM, you can
finally download and build the media assets for Zenodo. There is a script
which does that:

.. code-block:: console

   (zenodo)$ ./scripts/setup-assets.sh

Running services
~~~~~~~~~~~~~~~~

To run Zenodo locally, you will need to have some services running on your
machine.
At minimum you must have PostgreSQL, Elasticsearch 2.x, Redis and RabbitMQ.
You can either install all of those from your system package manager and run
them directly or better - use the provided docker image as before.

**The docker image is the recommended method for development.**

.. note::

   If you run the services locally, make sure you're running
   Elasticsearch **2.x**. Elasticsearch **5.x** is NOT yet supported.


To run only the essential services using docker, execute the following:

.. code-block:: console

    $ cd ~/src/zenodo
    $ docker-compose up -d

This should bring up four docker nodes with PostgreSQL (db), Elasticsearch (es),
RabbitMQ (mq), and Redis (cache). Keep this shell session alive.

Initialization
~~~~~~~~~~~~~~
Now that the services are running, it's time to initialize the Zenodo database
and the Elasticsearch index.

Create the database, Elasticsearch indices, messages queues and various
fixtures for licenses, grants, communities and users in a new shell session:

.. code-block:: console

   $ cd ~/src/zenodo
   $ workon zenodo
   (zenodo)$ ./scripts/init.sh

Let's also run the Celery worker on a different shell session:

.. code-block:: console

   $ cd ~/src/zenodo
   $ workon zenodo
   (zenodo)$ celery worker -A zenodo.celery -l INFO --purge

.. note::

    Here we assume all four services (db, es, mq, cache) are bound to localhost
    (see `zenodo/config.py <https://github.com/zenodo/zenodo/blob/master/zenodo/config.py/>`_).
    If you fail to connect those services, it is likely
    you are running docker through ``docker-machine`` and those services are
    bound to other IP addresses. In this case, you can redirect localhost ports
    to docker ports as follows.

    ``ssh -L 6379:localhost:6379 -L 5432:localhost:5432 -L 9200:localhost:9200 -L 5672:localhost:5672 docker@$(docker-machine ip)``

    The problem usually occurs among Mac and Windows users. A better solution
    is to install the native apps `Docker for Mac <https://docs.docker.com/docker-for-mac/>`_
    or `Docker for Windows <https://docs.docker.com/docker-for-windows/>`_
    (available since Docker v1.12) if possible,
    which binds docker to localhost by default.

Loading data
~~~~~~~~~~~~

Next, let's load some external data (only licenses for the time being). Loading
of this demo data is done asynchronusly with Celery, but depends on internet
access since it involves harvesting external OAI-PMH or REST APIs.

Make sure you keep the session with Celery worker alive. Launch the data
loading commands in a separate shell:

.. code-block:: console

   $ cd ~/src/zenodo
   $ workon zenodo
   (zenodo)$ zenodo opendefinition loadlicenses -s opendefinition
   (zenodo)$ zenodo opendefinition loadlicenses -s spdx
   (zenodo)$ ./scripts/index.sh

Finally, run the Zenodo development server in debug mode. You can do that by
setting up the environment flag:

.. code-block:: console

    (zenodo)$ export FLASK_DEBUG=True
    (zenodo)$ zenodo run

If you go to http://localhost:5000, you should see an instance of Zenodo,
similar to the production instance at https://zenodo.org.

Badges
~~~~~~
In order for the DOI badges to work you must have the Cairo SVG library and the
DejaVu Sans font installed on your system. Please see `Invenio-Formatter
<http://pythonhosted.org/invenio-formatter/installation.html>`_ for details.



You can find the original installation file `here <https://github.com/AJLite/zenodo/blob/master/INSTALL.rst/>`_ 
