Deploy
======

This section describes how we can use Rive ServiceMaker tools to deploy trained models for Riva AI Serives.

.. note::

    This section assumes you have completed :ref:`training` beforehand.

1. Deploy models
----------------

To deploy trained model :file:`exported-model.rmir`, create a directory to store the rmir file obtained after the training.

.. code-block:: bash

    $ mkdir -p your/path/toriva_quickstart_v1.10.0-beta/models/rmir
    $ cp <local results folder>/text_classification/text_classification_<version>/export_riva/exported-model.rmir \
            your/path/to/riva_quickstart_v1.10.0-beta/models/rmir

To generate the model repository, run the below code:

.. code-block:: bash

    $ bash your/path/to/riva_quickstart_v1.10.0-beta/riva_init.sh

Now by running :file:`riva_init.sh`, the model repository will be generated automatically at :file:`your/path/toriva_quickstart_v1.10.0-beta/models/models`

2. Riva Server Configuration
----------------------------

In Riva Skills Quick Start directory, let's enable deployed models:

.. code-block:: bash

    $ vim config.sh

* set :code:`service_enabled_asr=false`
* set :code:`service_enabled_nlp=true`
* set :code:`service_enabled_tts=false`
* set :code:`riva_model_loc=<your/path/to/riva_quickstart_v1.10.0-beta>/models`
* set :code:`use_existing_rmirs=true`

3. Start Riva Server
--------------------

Done! Now we finished our server configuration, let's start Riva Server:

.. code-block:: bash

    $ bash riva_start.sh
