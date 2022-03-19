Deploy
======

This section describes how we can use Rive ServiceMaker tools to deploy trained models for Riva AI Serives.

.. note::

    This section assumes you have completed :ref:`training` beforehand.

1. Launch RivaService Container
-------------------------------

To deploy trained model :file:`exported-model.rmir`, we need to first launch RivaService Container:

.. code-block:: bash

    $ cd riva_quickstart:1.10.0-beta
    $ mkdir rmirs
    $ cp <local results folder>/text_classification/text_classification_<version>/export_riva/exported-model.rmir rmirs/
    $ sudo docker run -it --rm --gpus all -v$(pwd):/data nvcr.io/nvidia/riva/riva-speech:1.10.0-beta-servicemaker

2. Deploy models
----------------

Inside the container, use :code:`riva-deploy` command to deploy :code:`.rmir` model to Riva model repository:

.. code-block:: bash

    $ mkdir /data/models
    $ riva-deploy -f /data/rmirs/exported-model.rmir:tlt_encode /data/models

When deployment is done, exit the RivaSerivce container.

3. Riva Server Configuration
----------------------------

In Riva Skills Quick Start directory, let's enable deployed models:

.. code-block:: bash

    $ vim config.sh

* set :code:`service_enabled_asr=false`
* set :code:`service_enabled_nlp=true`
* set :code:`service_enabled_tts=false`
* set :code:`riva_model_loc=<path/to/riva_quickstart_v1.10.0-beta>`
* set :code:`use_existing_rmirs=true`

4. Start Riva Server
--------------------

Done! Now we finished our server configuration, let's start Riva Server:

.. code-block:: bash

    $ bash riva_start.sh
