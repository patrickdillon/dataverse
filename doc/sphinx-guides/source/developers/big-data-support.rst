Big Data Support
================

Big data support is highly experimental. Eventually this content will move to the Installation Guide.

.. contents:: |toctitle|
        :local:

Various components need to be installed and configured for big data support.

Data Capture Module (DCM)
-------------------------

Data Capture Module (DCM) is an experimental component that allows users to upload large datasets via rsync over ssh.

Install a DCM
~~~~~~~~~~~~~

Installation instructions can be found at https://github.com/sbgrid/data-capture-module . Note that a shared filesystem between Dataverse and your DCM is required. You cannot use a DCM with non-filesystem storage options such as Swift.

Once you have installed a DCM, you will need to configure two database settings on the Dataverse side. These settings are documented in the :doc:`/installation/config` section of the Installation Guide:

- ``:DataCaptureModuleUrl`` should be set to the URL of a DCM you installed.
- ``:UploadMethods`` should be set to ``dcm/rsync+ssh``.
  
This will allow your Dataverse installation to communicate with your DCM, so that Dataverse can download rsync scripts for your users.

Downloading rsync scripts via Dataverse API
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The rsync script can be downloaded from Dataverse via API using an authorized API token. In the curl example below, substitute ``$PERSISTENT_ID`` with a DOI or Handle:

``curl -H "X-Dataverse-key: $API_TOKEN" $DV_BASE_URL/api/datasets/:persistentId/dataCaptureModule/rsync?persistentId=$PERSISTENT_ID``

How a DCM reports checksum success or failure to Dataverse
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Once the user uploads files to a DCM, that DCM will perform checksum validation and report to Dataverse the results of that validation. The DCM must be configured to pass the API token of a superuser. The implementation details, which are subject to change, are below.

The JSON that a DCM sends to Dataverse on successful checksum validation looks something like the contents of :download:`checksumValidationSuccess.json <../_static/installation/files/root/big-data-support/checksumValidationSuccess.json>` below:

.. literalinclude:: ../_static/installation/files/root/big-data-support/checksumValidationSuccess.json
   :language: json

- ``status`` - The valid strings to send are ``validation passed`` and ``validation failed``.
- ``uploadFolder`` - This is the directory on disk where Dataverse should attempt to find the files that a DCM has moved into place. There should always be a ``files.sha`` file and a least one data file. ``files.sha`` is a manifest of all the data files and their checksums. The ``uploadFolder`` directory is inside the directory where data is stored for the dataset and may have the same name as the "identifier" of the persistent id (DOI or Handle). For example, you would send ``"uploadFolder": "DNXV2H"`` in the JSON file when the absolute path to this directory is ``/usr/local/glassfish4/glassfish/domains/domain1/files/10.5072/FK2/DNXV2H/DNXV2H``.
- ``totalSize`` - Dataverse will use this value to represent the total size in bytes of all the files in the "package" that's created. If 360 data files and one ``files.sha`` manifest file are in the ``uploadFolder``, this value is the sum of the 360 data files.


Here's the syntax for sending the JSON.

``curl -H "X-Dataverse-key: $API_TOKEN" -X POST -H 'Content-type: application/json' --upload-file checksumValidationSuccess.json $DV_BASE_URL/api/datasets/:persistentId/dataCaptureModule/checksumValidation?persistentId=$PERSISTENT_ID``


Steps to set up a DCM mock for Development
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Install Flask.


Download and run the mock. You will be cloning the https://github.com/sbgrid/data-capture-module repo.

- ``git clone git://github.com/sbgrid/data-capture-module.git``
- ``cd data-capture-module/api``
- ``./dev_mock.sh``

If you see an error about not having Flask installed, install it as explained below.

On Mac, you can install Flask with:

- ``mkvirtualenv mockdcm``
- ``pip install -r requirements-mock.txt``

On Ubuntu/Debian, you can install Flask with:

- ``sudo apt install python-pip`` (will install python as well)
- ``pip install flask``

Once you have Flask installed, try running the dev mock script again:

- ``./dev_mock.sh``

This should spin up the DCM mock on port 5000.

Add Dataverse settings to use mock (same as using DCM, noted above):

- ``curl http://localhost:8080/api/admin/settings/:DataCaptureModuleUrl -X PUT -d "http://localhost:5000"``
- ``curl http://localhost:8080/api/admin/settings/:UploadMethods -X PUT -d "dcm/rsync+ssh"``

At this point you should be able to download a placeholder rsync script. Dataverse is then waiting for new from the DCM about if checksum validation has succeeded or not. First, you have to put files in place, which is usually the job of the DCM. You should substitute "X1METO" for the "identifier" of the dataset you create. You must also use the proper path for where you store files in your dev environment.

- ``mkdir /usr/local/glassfish4/glassfish/domains/domain1/files/10.5072/FK2/X1METO``
- ``mkdir /usr/local/glassfish4/glassfish/domains/domain1/files/10.5072/FK2/X1METO/X1METO``
- ``cd /usr/local/glassfish4/glassfish/domains/domain1/files/10.5072/FK2/X1METO/X1METO``
- ``echo "hello" > file1.txt``
- ``shasum file1.txt > files.sha``

Now the files are in place and you need to send JSON to Dataverse with a success or failure message as described above. Make a copy of ``doc/sphinx-guides/source/_static/installation/files/root/big-data-support/checksumValidationSuccess.json`` and put the identifier in place such as "X1METO" under "uploadFolder"). Then use curl as described above to send the JSON.

Troubleshooting
~~~~~~~~~~~~~~~

The following low level command should only be used when troubleshooting the "import" code a DCM uses but is documented here for completeness.

``curl -H "X-Dataverse-key: $API_TOKEN" -X POST "$DV_BASE_URL/api/batch/jobs/import/datasets/files/$DATASET_DB_ID?uploadFolder=$UPLOAD_FOLDER&totalSize=$TOTAL_SIZE"``

Repository Storage Abstraction Layer (RSAL)
-------------------------------------------

Configuring the RSAL Mock
~~~~~~~~~~~~~~~~~~~~~~~~~

Info for configuring the RSAL Mock: https://github.com/sbgrid/rsal/tree/master/mocks

Also, to configure Dataverse to use the new workflow you must do the following (see also the section below on workflows):

1. Configure the RSAL URL:

``curl -X PUT -d 'http://<myipaddr>:5050' http://localhost:8080/api/admin/settings/:RepositoryStorageAbstractionLayerUrl``

2. Update workflow json with correct URL information:

Edit internal-httpSR-workflow.json and replace url and rollbackUrl to be the url of your RSAL mock.

3. Create the workflow:

``curl http://localhost:8080/api/admin/workflows -X POST --data-binary @internal-httpSR-workflow.json -H "Content-type: application/json"``

4. List available workflows:

``curl http://localhost:8080/api/admin/workflows``

5. Set the workflow (id) as the default workflow for the appropriate trigger:

``curl http://localhost:8080/api/admin/workflows/default/PrePublishDataset -X PUT -d 2``

6. Check that the trigger has the appropriate default workflow set:

``curl http://localhost:8080/api/admin/workflows/default/PrePublishDataset``

7. Add RSAL to whitelist

8. When finished testing, unset the workflow:

``curl -X DELETE http://localhost:8080/api/admin/workflows/default/PrePublishDataset``

Configuring download via rsync
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In order to see the rsync URLs, you must run this command:

``curl -X PUT -d 'rsal/rsync' http://localhost:8080/api/admin/settings/:DownloadMethods``

TODO: Document these in the Installation Guide once they're final.

To specify replication sites that appear in rsync URLs:

``curl http://localhost:8080/api/admin/settings/:ReplicationSites -X PUT -d "dv.sbgrid.org:Harvard Medical School:USA,sbgrid.icm.uu.se:Uppsala University:Sweden,sbgrid.ncpss.org:Institut Pasteur de Montevideo:Uruguay,sbgrid.ncpss.org:Shanghai Institutes for Biological Sciences:China"``

In the GUI, this is called "Local Access". It's where you can compute on files on your cluster.

``curl http://localhost:8080/api/admin/settings/:LocalDataAccessPath -X PUT -d "/programs/datagrid"``

Workflows
---------

Dataverse can perform two sequences of actions when datasets are published: one prior to publishing (marked by a ``PrePublishDataset`` trigger), and one after the publication has succeeded (``PostPublishDataset``). The pre-publish workflow is useful for having an external system prepare a dataset for being publicly accessed (a possibly lengthy activity that requires moving files around, uploading videos to a streaming server, etc.), or to start an approval process. A post-publish workflow might be used for sending notifications about the newly published dataset.

Workflow steps are created using *step providers*. Dataverse ships with an internal step provider that offers some basic functionality, and with the ability to load 3rd party step providers. This allows installations to implement functionality they need without changing the Dataverse source code.

Steps can be internal (say, writing some data to the log) or external. External steps involve Dataverse sending a request to an external system, and waiting for the system to reply. The wait period is arbitrary, and so allows the external system unbounded operation time. This is useful, e.g., for steps that require human intervension, such as manual approval of a dataset publication.

The external system reports the step result back to dataverse, by sending a HTTP ``POST`` command to ``api/workflows/{invocation-id}``. The body of the request is passed to the paused step for further processing.

If a step in a workflow fails, Dataverse make an effort to roll back all the steps that preceeded it. Some actions, such as writing to the log, cannot be rolled back. If such an action has a public external effect (e.g. send an EMail to a mailing list) it is advisable to put it in the post-release workflow.

.. tip::
  For invoking external systems using a REST api, Dataverse's internal step
  provider offers a step for sending and receiving customizable HTTP requests.
  It's called *http/sr*, and is detailed below.

Administration
~~~~~~~~~~~~~~

A Dataverse instance stores a set of workflows in its database. Workflows can be managed using the ``api/admin/workflows/`` endpoints of the :doc:`/api/native-api`. Sample workflow files are available in ``scripts/api/data/workflows``.

At the moment, defining a workflow for each trigger is done for the entire instance, using the endpoint ``api/admin/workflows/default/«trigger type»``.

In order to prevent unauthorized resuming of workflows, Dataverse maintains a "white list" of IP addresses from which resume requests are honored. This list is maintained using the ``/api/admin/workflows/ip-whitelist`` endpoint of the :doc:`/api/native-api`. By default, Dataverse honors resume requests from localhost only (``127.0.0.1;::1``), so set-ups that use a single server work with no additional configuration.


Available Steps
~~~~~~~~~~~~~~~

Dataverse has an internal step provider, whose id is ``:internal``. It offers the following steps:

log
+++

A step that writes data about the current workflow invocation to the instance log. It also writes the messages in its ``parameters`` map.

.. code:: json

  {
     "provider":":internal",
     "stepType":"log",
     "parameters": {
         "aMessage": "message content",
         "anotherMessage": "message content, too"
     }
  }


pause
+++++

A step that pauses the workflow. The workflow is paused until a POST request is sent to ``/api/workflows/{invocation-id}``.

.. code:: json

  {
      "provider":":internal",
      "stepType":"pause"
  }


http/sr
+++++++

A step that sends a HTTP request to an external system, and then waits for a response. The response has to match a regular expression specified in the step parameters. The url, content type, and message body can use data from the workflow context, using a simple markup language. This step has specific parameters for rollback.

.. code:: json

  {
    "provider":":internal",
    "stepType":"http/sr",
    "parameters": {
        "url":"http://localhost:5050/dump/${invocationId}",
        "method":"POST",
        "contentType":"text/plain",
        "body":"START RELEASE ${dataset.id} as ${dataset.displayName}",
        "expectedResponse":"OK.*",
        "rollbackUrl":"http://localhost:5050/dump/${invocationId}",
        "rollbackMethod":"DELETE ${dataset.id}"
    }
  }

Available variables are:

* ``invocationId``
* ``dataset.id``
* ``dataset.identifier``
* ``dataset.globalId``
* ``dataset.displayName``
* ``dataset.citation``
* ``minorVersion``
* ``majorVersion``
* ``releaseStatus``
