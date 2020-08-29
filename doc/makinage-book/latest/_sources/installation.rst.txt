Installation
==============


Using Python Package Index
---------------------------

The releases of RxSci and Maki Nage are published on `pypi <https://pypi.org>`_. You can
install them with pip:

.. code:: console

    python3 -m pip install makinage


You can also install RxSci only if you want to use only it:

.. code:: console

    python3 -m pip install rxsci


Docker Images
--------------

Development Kafka server
.........................

for development purposes, you can use a development Kafka server running on a
single machine. A docker compose configuration is available to use it easily:

.. code:: console

    git clone https://github.com/maki-nage/docker.git
    cd docker/compose/mn-dev/
    docker-compose up -d kafka

The Kafka service is binded on **all** network interfaces of the machine, allowing for direct usage.

.. warning::

    Do not use this image on a non trusted network since anybody can access this
    kafka cluster. If you need to work on localhost only, you cna change the
    docker-compose configuration file.
