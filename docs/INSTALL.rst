Installation
============

See `README.rst <../README.rst>`_ for high level information about how to navigate the full documentation.

Prerequisites
-------------

At a minimum, you will need `Docker <https://docs.docker.com/install/>`_ and `docker-compose <https://docs.docker.com/compose/install/>`_.  You also will need a copy of this repository, either obtained via ``git`` or downloaded directly.  If you have ``git`` installed, this can be::

    git clone https://github.com/DigitalSlideArchive/DSA-WSI-DeID.git

Install commands need to be run from the ``devops/wsi_deid`` directory.  Examples are given via a command prompt, but a desktop version of Docker will work as well.

To download the repository directly, go to the `Releases Page <https://github.com/DigitalSlideArchive/DSA-WSI-DeID/releases>`_ and select the source code zip or tar.gz file for the desired version, or download the `master branch <https://github.com/DigitalSlideArchive/DSA-WSI-DeID/archive/master.zip>`_.

The software is known to work with a variety of operating systems:

- Windows 10 with Windows Subsystem for Linux (WSL) installed and Docker Desktop

- Ubuntu (version 16.04 and later)

- CentOS (version 7 and later)

- Windows Server 2019.  This operating system require some additional changes beyond the default install configuration, see `below for an example and comments <#windows-server-2019>`__ on using WS 2019.

Import and Export Paths
-----------------------

If you want to import and export data from your local filesystem into the Pilot, you will need to set up import and export paths, by mounting specific directories for import and export of files.  This set up is most readily done by creating a secondary docker-compose yaml file in the ``devops/wsi_deid`` directory, named ``docker-compose.local.yml`` which contains the following content (note that Windows Server 2019 may require changes to this file, see `below for an example and comments <#windows-server-2019>`__ on using WS 2019)::

    ---
    version: '3'
    services:
      girder:
        # Change "stable" to a version number (e.g., dsarchive/wsi_deid:v1.0.0)
        # to use a fixed version
        image: dsarchive/wsi_deid:stable
        volumes:
          - c:\NCI_WSI:/import
          - c:\DeID_WSI:/export

where the first part of the last two lines are paths on the local system that should be mounted into the ``import`` and ``export`` paths of the Pilot system, i.e. ``c:\wsi_deid\import:/import`` specifies that the local filesystem directory ``c:\wsi_deid\import`` is mounted into the Pilot as the ``/import`` path.  To use these defined import and export paths, instead of typing ``docker-compose up -d``, type::

    docker-compose -f docker-compose.yml -f docker-compose.local.yml up -d

which will extend and override the definitions in ``docker-compose.yml`` with those in ``docker-compose.local.yml``.

Initial Start
-------------

From a command prompt in the ``devops/wsi_deid`` directory, if you are using import and export paths, type::

    docker-compose pull
    docker-compose -f docker-compose.yml -f docker-compose.local.yml up -d

or without the import and export paths, type::

    docker-compose pull
    docker-compose up -d


This set up will download some necessary files (pre-built docker images) and start the system.  The database, local files, and some logs are stored in docker volumes.

The system will be available from a web browser on http://localhost:8080.

Note: If you prefer a different locally mounted port, you can specific that via an ENV VAR ``DSA_PORT``, e.g.::

    DSA_PORT=8888 docker-compose -f docker-compose.yml -f docker-compose.local.yml up -d

Update an Existing System
-------------------------

From a command prompt in the ``devops/wsi_deid`` directory, if you are using import and export paths, type::

    git pull
    docker-compose pull
    docker-compose down
    docker-compose -f docker-compose.yml -f docker-compose.local.yml up -d

or without the import and export paths, type::

    git pull
    docker-compose pull
    docker-compose down
    docker-compose up -d


This uses ``git`` to update the repository, fetches the latest build from docker, stops the currently running version, and starts the new version.

Complete Reset
~~~~~~~~~~~~~~

Information about images is stored in a persistent database located in a docker volume.  Processed images are stored in a second docker volume.  When a system is updated, this data persists.  To reset the system completely, deleting all information including users and processed images, first stop the system via ``docker-compose down``, then delete the docker volumes via the command ``docker volume rm wsi_deid_dbdata wsi_deid_fsdata wsi_deid_logs``.

Using a Specific Version
------------------------

By default, `docker-compose up` will use the most recent stable version of the software.  To use a specific version (e.g., `v1.0.0`), make sure you switch to that version from GitHub::

    git checkout v1.0.0

Modify the version in your ``docker-compose.local.yml`` file.  For example, change the line which reads ``image: dsarchive/wsi_deid:stable`` to ``image: dsarchive/wsi_deid:v1.0.0``.  Now, when you do::

    docker-compose pull
    docker-compose -f docker-compose.yml -f docker-compose.local.yml up -d

that version will be pulled and run.

For testing the latest docker image or a local docker image, remove the version from the image (e.g., ``image: dsarchive/wsi_deid``).  You can build the docker image locally by executing ``docker build --force-rm -t dsarchive/wsi_deid .`` in the top directory of the repository.

Debugging
---------

You can access logs of specific docker containers via::

    docker-compose logs

There are more detailed logs for the main container that can be viewed via::

    docker-compose exec girder cat /logs/info.log

You can follow the logs and see them update as they change::

    docker-compose logs -f
    docker-compose exec girder tail -F /logs/info.log

Fixing Common Problems
----------------------

If you accidentally delete one of the ``WSI DeID`` collection folders, simply restart the system with::

    docker-compose down
    docker-compose up

substituting whichever specific ``docker-compose up`` variant you normally use to run the system. This system restart will automatically recreate any of the ``WSI DeID`` collection folders that are tied to specific workflow states.

Admin User
----------

By default, when the system is first installed, there is one user with Administrator status with a default username of ``admin`` and password of ``password``.  It is strongly recommended that this be changed immediately, either by logging in and changing the password or by logging in, creating a new admin user and deleting the existing one.

Windows Server 2019
===================

There are several versions of Docker available on Windows Server 2019.  The exact version and manner of installation can affect how the software is installed.  Once Docker and docker-compose are installed, the software can start, though there may need to be changes to the ``docker-compose.local.yml`` file.

An example configuration file is provided, see `docker-compose.example-ws2019.local.yml <../devops/wsi_deid/docker-compose.example-ws2019.local.yml>`__.  There are some common issues that can occur which require uncommenting specific lines in the example file:

- If you see an error that includes ``invalid volume specification: 'wsi_deid_dbdata:/data/db:rw'``, uncomment the line that begins with ``image: mongo@sha256:``.  This error occurs because Docker is trying to use a Windows image for part of the system and linux images for other parts.  Uncommenting the line forces Docker to use a specific linux image of the mongo database.

- If after starting, mongo stops immediately (the command ``docker-compose logs`` will include a message containing ``aborting after fassert() failure``), uncomment the line beginning with ``command: "bash -c 'mongod``.

Example Installation on WS 2019
-------------------------------

As an example of installing the software on a fresh install of Windows Server 2019 (tested on version 1809, OS Build 17763.737), the following powershell commands were used.

Install Docker::

    Install-Module DockerProvider
    Install-Package Docker -ProviderName DockerProvider -RequiredVersion preview

Enable linux images in docker::

    [Environment]::SetEnvironmentVariable("LCOW_SUPPORTED", "1", "Machine")

Restart the server::

    shutdown /r

Once it has restarted, ensure the docker service is running and install docker-compose::

    Restart-Service docker
    [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
    Invoke-WebRequest https://github.com/docker/compose/releases/download/1.27.4/docker-compose-Windows-x86_64.exe -UseBasicParsing -OutFile $Env:ProgramFiles\Docker\docker-compose.exe

Install our software::

    mkdir c:\project
    Invoke-WebRequest https://github.com/DigitalSlideArchive/DSA-WSI-DeID/archive/master.zip -outfile c:\project\dsa.zip
    Expand-Archive -LiteralPath c:\project\dsa.zip -DestinationPath c:\project
    cd c:\project\DSA-WSI-DeID-master\devops\wsi_deid
    copy docker-compose.example-ws2019.local.yml docker-compose.local.yml
    
If needed, edit ``docker-compose.local.yml``.  For this installation. the ``command:`` line was uncommented.

Start the software::

    docker-compose -f docker-compose.yml -f docker-compose.local.yml up -d

Sample Data
===========

A small set of sample WSI files and a sample DeID Upload excel file are available on `data.kitware.com <https://data.kitware.com/#item/5f87213d50a41e3d19ea89c2>`_.

`Download a zip file of the sample files. <https://data.kitware.com/api/v1/file/5f87213d50a41e3d19ea89c4/download>`_

Redaction Business Rules
========================

Some metadata fields are automatically modified by default.  For example, certain dates are converted to always be January 1st of the year of the original date.  Embedded titles and filenames are replaced with a specified Image ID.  Some of these modifications vary by WSI vendor format.

To modify these business rules, it is recommended that this repository is forked or an additional python module is created that alters the ``get_standard_redactions`` function and the vendor-specific variations of that function (e.g., ``get_standard_redactions_format_aperio``) located in the [process.py](https://github.com/DigitalSlideArchive/DSA-WSI-DeID/blob/master/wsi_deid/process.py) source file.
