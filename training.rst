.. _training:

Training
========

This section describes how we can use TAO toolkit to train text classification models which enables Virtual Assistant to answer questions from queries.

1. TAO Mounts
-------------
Create :file:`.tao_mounts.json` to configure the mount destinations for TAO toolkit.

.. code:: bash

    $ vim ~/.tao_mounts.json

Specify :code:`"Mounts"` in :file:`.tao_mounts.json`:

.. code:: json

    {
            "Mounts": [
                    {
                            "source": <local data folder>,
                            "destination": "/data"
                    },
                    {
                            "source": <local specs folder>,
                            "destination": "/specs"
                    },
                    {
                            "source": <local results folder>,
                            "destination": "/results"
                    },
                    {
                            "source": <local folder/.cache>,
                            "destination": "/root/.cache"
                    }
            ],
            "DockerOptions": {
                    "shm_size": "16G",
                    "ulimits": {
                            "memlock": -1,
                            "stack": 67108864
                    }
            }
    }

2. Specification files
----------------------
Download specification files which contain parameter configurations for subtasks in the training pipeline, including training, evaluating, and exporting the model.

.. code-block:: bash

    $ tao text_classification download_specs -r /results/text_classification/default_specs/ \ 
                                             -o /specs/nlp/text_classification

3. Data Preparation
-------------------

:file:`nlu.yml` stores the training data of text classification model, which has the format like this:

.. code-block:: yaml

    nlu:
        - intent: intent1
        examples: |
            - utterance 1
            - utterance 2
            - utterance 3
        
        - intent: intent2
        examples: |
            - utterance 1
            - utterance 2
            - utterance 3

        ....

Create file *nlu.yml* and copy the content of :ref:`nluyml` under :file:`<local data folder>/domain/text_classification/`:

.. code:: bash

    $ mkdir -p <local data folder>/domain/text_classification/
    $ vim <local data folder>/domain/text_classification/nlu.yml


    
Feel free to add your intents and sample data into your own :file:`nlu.yml`.

.. _data_convert:

4. Data Format Conversion
-------------------------
:ref:`convertyaml` converts :file:`nlu.yml` into TAO training format. For simplicity, copy :ref:`convertyaml` this script at the same location with :file:`nlu.yml`.

The syntax of :file:`convert_yaml.py` is as follows:

.. code-block:: bash

    $ python3 convert_yaml.py --dd <local data folder> --yf <folder to store nlu.yml> --tr <train-val-split ratio> --shuffle

Use :file:`convert_yaml.py` to convert our :file:`nlu.yml`:

.. code-block:: bash

    $ python3 convert_yaml.py --dd <local data folder> --yf domain/text_classification --tr 0.8 --shuffle

You should see :file:`train.tsv`, :file:`val.tsv`, and :file:`labels.csv` are generated in the current folder.

.. _train_config:

5. Training configurations
--------------------------
:file:`config_chatbot.sh` contains the flags that will be passed to launch TAO Toolkit training. Let's create this script under the folder of Riva Skills Quick Start. 

.. code:: bash

    $ cd riva_quickstart:1.10.0-beta
    $ vim config_chatbot.sh

:file:`config_chatbot.sh`:

.. code:: text

    RIVA_MOUNTED_DATA_DIR='/data'
    RIVA_MOUNTED_SPECS_DIR='/specs'
    RIVA_MOUNTED_RESULTS_DIR='/results'
    LOCAL_RESULT_DIR=<local results folder>
    RIVA_REPO="path/to/riva_quickstart_v1.10.0-beta"
    RIVA_SERVICE_MAKER=nvcr.io/nvidia/riva/riva-speech:1.10.0-beta-servicemaker
    NUM_CLASSES=<number of intents in training set>
    NUM_EPOCH=500
    NGC_API_KEY=<Your NGC API Key>
    ENCRYPTION_KEY="tlt_encode"
    GPUS=1

* :file:`RIVA_MOUNTED_DATA_DIR`, :file:`RIVA_MOUNTED_SPECS_DIR`, :file:`RIVA_MOUNTED_RESULTS_DIR` are defined in :file:`~/.tao_mounts.json`, which are to be referenced by TAO toolkit during training.
* Replace :file:`NGC_API_KEY` with your NGC API Key.

6. Model training
-----------------

:ref:`train` contains the model training pipelines. Copy :ref:`train` under the folder of Riva Skills Quick Start.

Execute :file:`train.sh` to start training:

.. code-block:: bash

    $ bash train.sh

When training is done, we should find exported :file:`exported-model.riva` and :file:`exported-model.rmir` under :file:`<local results folder>/text_classification/text_classification_<version>/export_riva`
