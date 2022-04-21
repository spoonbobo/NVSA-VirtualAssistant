References
==========

.. _nluyml:

nlu.yml
-------

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

        - intent: ask_dgx_a100_nvme
          examples: |
              - dgxa100 system nvme
              - dgxa100 ssds
              - dgxa100 read data speed
              - dgx a100 system volume
              - dgxa100 hard disk
              - dgx a100 network data transfer

        - intent: ask_dgx_a100_software_stack
          examples: |
              - software stack dgx a100
              - dgx deep learning frameworks
              - SDK software dgx a100
              - NGC containers with dgx a100
              - dgxa100 DL containers
              - dgx a100 system ML containers
              - ML DL containers in dgd a100 system 

        - intent: ask_nvidia_container_toolkit
          examples: |
              - NVIDIA container toolkit
              - nv container toolkit
              - nvidia docker
              - NVIDIA container runtime library
              - docker leverage nvidia gpu
              - gpu accelerated docker
              - gpu docker

        - intent: ask_nvidia_cuda_toolkit
          examples: |
              - nvidia cuda toolkit
              - cuda 11
              - cuda 10
              - cuda toolkit
              - cuda liraris
              - create GPU-accelerated applications
              - accelerated applications

        - intent: ask_dgx_front_fan_replace
          examples: |
              - replace front fan dgx
              - check front fan status dgx
              - identify failed fan
              - change the front fan
              - add new fan module
              - fan working properly

        - intent: ask_dgx_power_supply
          examples: |
              - replace power supply dgx
              - identify failed power supply
              - how to get power supply?
              - how to replace power supply
              - how to remove and add power supply
              - failed power supply


.. _convertyaml:
      
convert_yaml.py
---------------

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

.. _train:

train.sh
--------

.. code-block:: shell

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

.. _apppy:

app.py
------

.. code-block:: python

    from flask import Flask, render_template, request, url_for, redirect
    from werkzeug.utils import secure_filename
    import os
    import soundfile as sf
    import librosa
    import wave
    from pydub import AudioSegment
    from dgx_enquire import dgx_enquire
    from dgx_enquire_audio import dgx_enquire_audio
    from dgx_resp import dgx_resp
    app = Flask(__name__)

    AUDIO_SAMPLE_RATE = 16000

    dgxWelcomeMsg = ["Hi, I'm DGX Bot. You can ask me questions about DGX systems. For example: ",
    " ",
    "What is DGX System?",
    "How to monitor the DGX System?",
    "What are the storage options available for DGX system?",
    "What are the benefits of using DGX?"]

    dgxData = {
        "queries": [None],
        "responses": [dgxWelcomeMsg],
        "count": 1,
        "reloadSpeech": False  # offset the reload.
    }

    @app.route("/")
    def root_page():
        return render_template('home.html')

    @app.route('/dgxSubmit', methods=['POST'])
    def dgxSubmit():
        dgx_query = request.form['dgx_query']
        print(dgx_query)
        if dgx_query == '':
            dgx_query = "Hello, DGX!"
        dgxData["queries"].append(dgx_query)
        dgxData["count"] += 1
        return redirect(url_for('dgxChat', query=dgx_query))

    @app.route('/dgxUpload', methods=['POST'])
    def dgxUpload():
        if "audio_file" in request.files:
            dgxAudioFile = request.files['audio_file']
            i = len(os.listdir("dgxAudios"))
            audioSavePath = "dgxAudios/audio_file_{}.wav".format(i)
            dgxAudioFile.save(audioSavePath)
            audio, sample_rate = librosa.load(audioSavePath)
            sf.write(audioSavePath, audio, sample_rate)
            wf = wave.open(audioSavePath, 'rb')
            with open(audioSavePath, 'rb') as fh:
                data = fh.read()
            # Accepted by Riva
            dgx_query = dgx_enquire_audio(wf, data)
            dgxData["queries"].append(dgx_query)
            dgxData["count"] += 1
            return redirect(url_for('dgxChat', query=dgx_query))

    @app.route('/DGXChat')
    def dgxChat():
        dgxQuery = dgxData["queries"][dgxData["count"] - 1]
        if dgxQuery != None:
            with open("dgxUtter/dgx_utter.txt", "a") as f:
                f.write(dgxQuery + "\n")
        if (dgxQuery != None) and (len(dgxData["responses"]) < dgxData["count"]):
            className = dgx_enquire([dgxQuery])
            dgxResp = dgx_resp(className)
            dgxData["responses"].append(dgxResp)
        print(dgxData)
        return render_template("DGXChat.html", dgxData=dgxData)

    if __name__ == '__main__':
        app.run()

.. _dgxenquire:

dgx_enquire.py
--------------

.. code-block:: python

    def dgx_enquire_audio(wf, data):
        channel = grpc.insecure_channel("10.19.27.126:50051")
        client = rasr_srv.RivaSpeechRecognitionStub(channel)
        config = rasr.RecognitionConfig(
            encoding=ra.AudioEncoding.LINEAR_PCM,
            sample_rate_hertz=wf.getframerate(),
            language_code="en-US",
            max_alternatives=1,
            enable_automatic_punctuation=False,
            audio_channel_count=1
        )

        request = rasr.RecognizeRequest(config=config, audio=data)
        response = client.Recognize(request)
        return response.results[0].alternatives[0].transcript

.. _dgxenquireaudio:

dgx_enquire_audio.py
--------------------

.. code-block:: python

    def dgx_enquire(queries):
        channel = grpc.insecure_channel("10.19.27.126:50051")
        riva_nlp = rnlp_srv.RivaLanguageUnderstandingStub(channel)
        request = rnlp.TextClassRequest()
        request.model.model_name = "riva_text_classification_default"

        for query in queries:
            request.text.append(query)

        resp = riva_nlp.ClassifyText(request)
        return resp.results[0].labels[0].class_name


.. _dgxchat:

DGXChat.html
------------

.. code-block:: html

    <!DOCTYPE html>
    <html lang="en">
        <head>
            <meta charset="UTF-8" />
            <meta http-equiv="X-UA-Compatible" content="IE=edge" />
            <meta name="viewport" content="width=device-width, initial-scale=1.0" />
            <link
                href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css"
                rel="stylesheet"
                integrity="sha384-1BmE4kWBq78iYhFldvKuhfTAU6auU8tT94WrHftjDbrCEXSU1oBoqyl2QvZ6jIW3"
                crossorigin="anonymous"
            />

            <link href="{{ url_for('static', filename='main.css') }}" rel="stylesheet" />

            <!-- Scrolling behaviour (Legacy) -->
            <script>
                function updateScroll() {
                    var element = document.getElementById('overlay-scroll')
                    element.scrollTop = element.scrollHeight
                }
                setInterval(updateScroll, 500)
            </script>

            <title>DGX Chatbot</title>
        </head>
        <body>
            <div class="container-fluid w-75 mh-75 py-4" style="background: rgba(0, 0, 0, 0) !important">
                <header class="pb-3 mb-4 border-bottom" style="background: rgba(0, 0, 0, 0) !important">
                    <a href="/" class="d-flex align-items-center text-dark text-decoration-none">
                        <img src="{{ url_for('static', filename='icon.png') }}" style="height: 2rem" />
                        <span class="fs-4 text-white" style="margin-left: 0.5rem">NVIDIA AI Hub</span>
                    </a>
                </header>
                <div class="p-5 bg-light mh-75 rounded-3" style="background: rgba(0, 0, 0, 0.5) !important">
                    <div
                        class="overflow-auto"
                        style="height: 35rem; scroll-behavior: smooth; display: flex; flex-direction: column-reverse"
                    >
                        <div class="container-fluid rounded mt-2">
                            {% for i in range(dgxData["count"]) %}
                            <!-- User queries -->
                            {% if dgxData["queries"][i] != None %}
                            <div class="row mb-3">
                                <div class="col">
                                    <div class="card float-end rouded" style="display: inline-block; max-width: 75%">
                                        <div class="card-body">
                                            <p class="card-text mb-0">{{ dgxData["queries"][i] }}</p>
                                        </div>
                                    </div>
                                </div>
                            </div>
                            {% endif %}

                            <!-- Chatbot responses -->
                            <div class="row mb-3">
                                <h5 class="text-white py-1">
                                    DGX Chatbot
                                    <svg
                                        xmlns="http://www.w3.org/2000/svg"
                                        width="16"
                                        height="16"
                                        fill="currentColor"
                                        class="bi bi-robot"
                                        viewBox="0 0 16 16"
                                    >
                                        <path
                                            d="M6 12.5a.5.5 0 0 1 .5-.5h3a.5.5 0 0 1 0 1h-3a.5.5 0 0 1-.5-.5ZM3 8.062C3 6.76 4.235 5.765 5.53 5.886a26.58 26.58 0 0 0 4.94 0C11.765 5.765 13 6.76 13 8.062v1.157a.933.933 0 0 1-.765.935c-.845.147-2.34.346-4.235.346-1.895 0-3.39-.2-4.235-.346A.933.933 0 0 1 3 9.219V8.062Zm4.542-.827a.25.25 0 0 0-.217.068l-.92.9a24.767 24.767 0 0 1-1.871-.183.25.25 0 0 0-.068.495c.55.076 1.232.149 2.02.193a.25.25 0 0 0 .189-.071l.754-.736.847 1.71a.25.25 0 0 0 .404.062l.932-.97a25.286 25.286 0 0 0 1.922-.188.25.25 0 0 0-.068-.495c-.538.074-1.207.145-1.98.189a.25.25 0 0 0-.166.076l-.754.785-.842-1.7a.25.25 0 0 0-.182-.135Z"
                                        />
                                        <path
                                            d="M8.5 1.866a1 1 0 1 0-1 0V3h-2A4.5 4.5 0 0 0 1 7.5V8a1 1 0 0 0-1 1v2a1 1 0 0 0 1 1v1a2 2 0 0 0 2 2h10a2 2 0 0 0 2-2v-1a1 1 0 0 0 1-1V9a1 1 0 0 0-1-1v-.5A4.5 4.5 0 0 0 10.5 3h-2V1.866ZM14 7.5V13a1 1 0 0 1-1 1H3a1 1 0 0 1-1-1V7.5A3.5 3.5 0 0 1 5.5 4h5A3.5 3.5 0 0 1 14 7.5Z"
                                        />
                                    </svg>
                                </h5>
                                <div class="col">
                                    <div class="card float-left rounded-2" style="display: inline-block; max-width: 75%">
                                        <div class="card-body" style="background-color: rgba(50, 205, 20, 0.3) !important">
                                            {% if dgxData["responses"][i] != None %} {% for resp in dgxData["responses"][i]
                                            %} {% if resp != None %} {% if resp != " " %}
                                            <p class="card-text mb-0">{{ resp }}</p>
                                            {% else %} <br />
                                            {% endif %} {% endif %} {% endfor %} {% else %}
                                            <p class="card-text mb-0">
                                                Sorry, I don't quite get what you asked. Can you rephrase?
                                            </p>
                                            {% endif %}
                                        </div>
                                    </div>
                                </div>
                            </div>
                            {% endfor %}
                        </div>
                    </div>

                    <div class="row mt-3">
                        <div class="col-md-11">
                            <form action="/dgxSubmit" method="post">
                                <div class="input-group input-group-lg">
                                    <input
                                        id="dgxQueryInput"
                                        type="text"
                                        name="dgx_query"
                                        placeholder="Hello, DGX!"
                                        class="form-control"
                                        aria-label="Sizing example input"
                                        aria-describedby="inputGroup-sizing-lg"
                                    />

                                    <!-- Send message -->
                                    <button
                                        class="btn btn-outline-secondary"
                                        type="submit"
                                        value="submit"
                                        id="button-addon2"
                                    >
                                        <svg
                                            xmlns="http://www.w3.org/2000/svg"
                                            width="16"
                                            height="16"
                                            fill="currentColor"
                                            class="bi bi-send-fill"
                                            viewBox="0 0 16 16"
                                        >
                                            <path
                                                fill-rule="evenodd"
                                                d="M15.964.686a.5.5 0 0 0-.65-.65L.767 5.855H.766l-.452.18a.5.5 0 0 0-.082.887l.41.26.001.002 4.995 3.178 3.178 4.995.002.002.26.41a.5.5 0 0 0 .886-.083l6-15Zm-1.833 1.89.471-1.178-1.178.471L5.93 9.363l.338.215a.5.5 0 0 1 .154.154l.215.338 7.494-7.494Z"
                                            />
                                        </svg>
                                    </button>
                                </div>
                            </form>
                        </div>
                        <div class="col-md-1">
                            <div class="input-group input-group-lg">
                                <button id="dgxAudioControl" class="btn btn-outline-secondary">
                                    <svg
                                        xmlns="http://www.w3.org/2000/svg"
                                        width="16"
                                        height="16"
                                        fill="currentColor"
                                        class="bi bi-mic"
                                        viewBox="0 0 16 16"
                                    >
                                        <path
                                            d="M3.5 6.5A.5.5 0 0 1 4 7v1a4 4 0 0 0 8 0V7a.5.5 0 0 1 1 0v1a5 5 0 0 1-4.5 4.975V15h3a.5.5 0 0 1 0 1h-7a.5.5 0 0 1 0-1h3v-2.025A5 5 0 0 1 3 8V7a.5.5 0 0 1 .5-.5z"
                                        />
                                        <path
                                            d="M10 8a2 2 0 1 1-4 0V3a2 2 0 1 1 4 0v5zM8 0a3 3 0 0 0-3 3v5a3 3 0 0 0 6 0V3a3 3 0 0 0-3-3z"
                                        />
                                    </svg>
                                </button>
                            </div>
                        </div>
                        <p id="micStatus" class="text-white py-3">
                            Speech recognition is in offline mode. Press
                            <svg
                                xmlns="http://www.w3.org/2000/svg"
                                width="16"
                                height="16"
                                fill="currentColor"
                                class="bi bi-mic"
                                viewBox="0 0 16 16"
                            >
                                <path
                                    d="M3.5 6.5A.5.5 0 0 1 4 7v1a4 4 0 0 0 8 0V7a.5.5 0 0 1 1 0v1a5 5 0 0 1-4.5 4.975V15h3a.5.5 0 0 1 0 1h-7a.5.5 0 0 1 0-1h3v-2.025A5 5 0 0 1 3 8V7a.5.5 0 0 1 .5-.5z"
                                />
                                <path
                                    d="M10 8a2 2 0 1 1-4 0V3a2 2 0 1 1 4 0v5zM8 0a3 3 0 0 0-3 3v5a3 3 0 0 0 6 0V3a3 3 0 0 0-3-3z"
                                />
                            </svg>
                            to start recording, and press it again to stop recording.
                        </p>
                        <span id="micMsg" class="text-muted mt-0"></span>
                    </div>

                    <div>
                        <script>
                            const uploadURL = "{{ url_for('dgxUpload') }}"
                            const downloadLink = document.getElementById('download')
                            const stopButton = document.getElementById('dgxAudioControl')
                            var microphoneStatus = 'Not Recording' // init state
                            document.getElementById('micMsg').innerHTML =
                                'Streaming recognition implementation is in progress. Current offline recognition may be slow, use it at your own risk'

                            const handleSuccess = function (stream) {
                                // microphoneStatus = !microphoneStatus
                                const options = { mimeType: 'audio/webm' }
                                let recordedChunks = []
                                let mediaRecorder = new MediaRecorder(stream, options)

                                mediaRecorder.addEventListener('dataavailable', function (e) {
                                    if (e.data.size > 0) recordedChunks.push(e.data)
                                })

                                mediaRecorder.addEventListener('stop', function () {
                                    let blob = new Blob(recordedChunks, { type: 'audio/webm' })
                                    let formData = new FormData()
                                    formData.append('audio_file', blob, 'audio_file.wav')
                                    fetch('/dgxUpload', { method: 'POST', body: formData, cache: 'no-cache' }).then(
                                        (resp) => {
                                            if (resp.status == 200) {
                                                window.location.reload(true)
                                            }
                                        }
                                    )
                                })
                                stopButton.addEventListener('click', function () {
                                    if (microphoneStatus == 'Recording') {
                                        mediaRecorder.stop()
                                        microphoneStatus = 'Not Recording'
                                        document.getElementById('micStatus').innerHTML =
                                            'Microphone Status: ' + microphoneStatus
                                    } else {
                                        mediaRecorder.start()
                                        microphoneStatus = 'Recording'
                                        document.getElementById('micStatus').innerHTML =
                                            'Microphone Status: ' + microphoneStatus
                                    }
                                })
                            }
                            navigator.mediaDevices.getUserMedia({ audio: true, video: false }).then(handleSuccess)
                        </script>
                    </div>
                </div>
                <footer class="pt-3 mt-4 text-muted border-top">&copy; 2021</footer>
            </div>
            <script
                src="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/js/bootstrap.bundle.min.js"
                integrity="sha384-ka7Sk0Gln4gmtz2MlQnikT1wXgYsOg+OMhuP+IlRH9sENBO0LRn5q+8nbTov4+1p"
                crossorigin="anonymous"
            ></script>
        </body>
    </html>