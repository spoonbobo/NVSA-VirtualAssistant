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

Create :file:`nlu.yml` under :file:`<local data folder>/domain/text_classification/`:

.. code:: bash

    $ mkdir -p <local data folder>/domain/text_classification/
    $ vim <local data folder>/domain/text_classification/nlu.yml

Let's add sample intents in :file:`nlu.yml`:

.. code-block:: yaml

    nlu:
        - intent: ask_dgx_a100_pcie_gen4
        examples: |
            - pci express dgx a100
            - PCIe bandth
            - PCI switch infrastructure
            - PCIe gen4
            - dgxa100 pcie
            - dgxa100 switch infrastructure

        - intent: ask_dgx_a100_m2_nvme_boot_replace
        examples: |
            - replace boot drive dgx a100
            - change drive for boot dgx a100
            - steps to replace boot drive for dgx a100
            - replace nvme for dgx a100 start up
            - m2 nvme on dgx a100 boot replace
            - detach dgx a100 boot drive

        - intent: ask_dgx_a100_upgrade_dimm
        examples: |
            - additional dimms dgx a100
            - add dimms dgx a100
            - 16 additional dual-inline memory modules
            - how to add dimms
            - how to upgrade diims
            - upgrade diims
            - extra dimms

        - intent: ask_dgx_air_gapped
        examples: |
            - dgx installation isolated from networks
            - air-gapped dgx systems
            - air-gapped for security dgx
            - isolated dgx systems
            - update software on airgapped dgx systems
            - how to update packages on dgx using over-the-network method
            - air gapped dgx over the network
    
Feel free to add your intents and sample data into your own :file:`nlu.yml`.

4. Data Format Conversion
-------------------------
:file:`convert_yaml.py` converts :file:`nlu.yml` into TAO training format. For simplicity, create this script at the same location with :file:`nlu.yml`.

:file:`convert_yaml.py`:

.. code-block:: python

    import os
    import yaml
    import re
    import random
    import argparse
    from yaml import load, dump
    try:
        from yaml import CLoader as Loader, CDumper as Dumper
    except ImportError:
        from yaml import Loader, Dumper

    def process_text_tc(x):
        x = x[2:]
        x = x.replace("[", "")
        x = x.replace("]", "")
        return re.sub("[\(\[].*?[\)\]]", "", x)

    def read_yaml(yamlFile, taskName):
        with open(yamlFile, "r") as f:
            data = yaml.safe_load(f)
        return data[taskName]

    def convert_yaml(dataDir, yamlFileList, resTrainPath, resValPath, labelPath, trainRatio, shuffleData):

        open(resTrainPath, "w").close()
        open(resValPath, "w").close()
        open(labelPath, "w").close()

        data = []
        for yamlFile in yamlFileList:
            yamlFilePath = os.path.join(dataDir, yamlFile, "nlu.yml")
            data += read_yaml(yamlFilePath, 'nlu')

        with open(resTrainPath, "a") as train_f:
            with open(resValPath, "a") as val_f:
                with open(labelPath, "a") as label_f:
                    trainData, valData, labels = [], [], []
                    for intentIdx in range(len(data)):
                        labels.append(data[intentIdx]["intent"])
                        examples = data[intentIdx]['examples'].split("\n")
                        examplesNoDash = list(map(process_text_tc, examples))[:-1]
                        numTrainData = int(len(examplesNoDash) * trainRatio)
                        for expIdx in range(len(examplesNoDash)):
                            line = examplesNoDash[expIdx] + '\t' + str(intentIdx)
                            if expIdx < numTrainData:
                                trainData.append(line)
                            else:
                                valData.append(line)
                    if shuffleData:
                        random.shuffle(trainData)
                        random.shuffle(valData)
                    label_f.write("\n".join(labels))
                    train_f.write("\n".join(trainData))
                    val_f.write("\n".join(valData))

    if __name__ == '__main__':
        parser = argparse.ArgumentParser()
        parser.add_argument("--dd", "--data_dir", help="path to data dir", type=str)
        parser.add_argument("--yf", "--yaml_file", nargs='+', help="path to yaml file", required=True)
        parser.add_argument("--tm", "--train_manifest", help="result train manifest", type=str)
        parser.add_argument("--vm", "--val_manifest", help="result val manifest", type=str)
        parser.add_argument("--lb", "--label_file", help="label file", type=str)
        parser.add_argument("--tr", "--train_ratio", help="train manifest ratio", type=float)
        parser.add_argument("--shuffle", help="shuffle data", action="store_true")

        args = parser.parse_args()

        if not args.dd:
            print("no data dir is specified. set current folder as default.")
            args.dd = os.getcwd()

        if not args.yf:
            print("you must provide at least 1 yaml file to use this script.")
            exit()

        if not args.tm:
            args.tm = "train.tsv"

        if not args.vm:
            args.vm = "val.tsv"

        if not args.lb:
            args.lb = 'labels.csv'

        if not args.tr:
            args.tr = 0.8

        if not args.shuffle:
            args.shuffle = False

        convert_yaml(args.dd, args.yf, args.tm, args.vm, args.lb, args.tr, args.shuffle)

The syntax of :file:`convert_yaml.py` is as follows:

.. code-block:: bash

    $ python3 convert_yaml.py --dd <local data folder> --yf <folder to store nlu.yml> --tr <train-val-split ratio> --shuffle

Use :file:`convert_yaml.py` to convert our :file:`nlu.yml`:

.. code-block:: bash

    $ python3 convert_yaml.py --dd <local data folder> --yf domain/text_classification --tr 0.8 --shuffle

You should see :file:`train.tsv`, :file:`val.tsv`, and :file:`labels.csv` are generated in the current folder. For TAO training format for text_classification task, please check `TAO Toolkit Text Classification <https://docs.nvidia.com/tao/tao-toolkit/text/nlp/text_classification.html>`_.

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
:file:`train.sh` contains the model training pipelines. Create this script under the folder of Riva Skills Quick Start.

.. code:: bash

    $ vim train.sh

:file:`train.sh`:

.. code:: bash

    #!/bin/bash
    temp_result_folder_path=$RIVA_MOUNTED_RESULTS_DIR/text_classification/text_classification_0
    temp_local_result_folder_path=$LOCAL_RESULT_DIR/text_classification/text_classification_0

    # 1. Check count for version
    version_count="$(ls "$LOCAL_RESULT_DIR/text_classification" | wc -l )"

    if [[ $version_count -ge 1 ]];
    then
            temp_result_folder_path=$RIVA_MOUNTED_RESULTS_DIR/text_classification/text_classification_$version_count
            temp_local_result_folder_path=$LOCAL_RESULT_DIR/text_classification/text_classification_$version_count
    fi

    # 2. TAO Toolkit: text classification task
    echo "The TAO-trained model will be stored at $temp_local_result_folder_path (docker: $temp_result_folder_path)"

    tao text_classification train \
            -e $RIVA_MOUNTED_SPECS_DIR/text_classification/train.yaml \
            -g $GPUS \
            -k $ENCRYPTION_KEY \
            -r $temp_result_folder_path \
            training_ds.file_path=$RIVA_MOUNTED_DATA_DIR/dgxChatbot/text_classification/train.tsv \
            validation_ds.file_path=$RIVA_MOUNTED_DATA_DIR/dgxChatbot/text_classification/val.tsv \
            model.class_labels.class_labels_file=$RIVA_MOUNTED_DATA_DIR/dgxChatbot/text_classification/labels.csv \
            model.dataset.num_classes=$NUM_CLASSES \
            trainer.max_epochs=$NUM_EPOCH

    # 3. TAO Toolkit: export text classification model to RIVA format
    tao text_classification export \
            -e "$RIVA_MOUNTED_SPECS_DIR/text_classification/export.yaml" \
            -g $GPUS \
            -m $temp_result_folder_path/checkpoints/trained-model.tlt \
            -k $ENCRYPTION_KEY \
            -r $temp_result_folder_path/export_riva \
            export_format=RIVA

    # 4. TAO Toolkit: export RIVA format model to rmir
    docker run --gpus all --rm -v $temp_local_result_folder_path/export_riva:/servicemaker-dev \
            -v $RIVA_REPO:/riva-repo --entrypoint="/bin/bash" \
            $RIVA_SERVICE_MAKER /riva-repo/build_rmir_nlp_tc.sh

Execute :file:`train.sh` to start training:

.. code-block:: bash

    $ bash train.sh

When training is done, we should find exported :file:`exported-model.riva` and :file:`exported-model.rmir` under :file:`<local results folder>/text_classification/text_classification_<version>/export_riva`