.. _frontend:

Frontend
========

There are number of frontends to access Riva AI Services. This sections demonstrates how we can create a DGX chatbot interface using `Flask <https://flask.palletsprojects.com/en/2.0.x/quickstart/>`_ web framework.

1. Prerequisites
----------------

Install flask using pip in your environment:

.. code-block:: bash

    $ pip install Flask

2. Peeking Flask App
--------------------

In this section, we are going to create a Flask app to deploy DGX Chatbot in following structure:

.. code-block:: bash

    .
    ├── app.py
    ├── dgx_enquire.py
    ├── dgx_resp.py
    ├── dgxUtter
    │   └── dgx_utter.txt
    ├── static
    │   ├── banner.jpg
    │   ├── bg1.jpg
    │   ├── chat_theme.css
    │   ├── dgxChat_arch.jpg
    │   ├── icon.png
    │   ├── main.css
    │   └── script.js
    ├── stream.js
    └── templates
        ├── DGXChat.html
        └── home.html

To create the sample web app, clone the sample app from Github:

.. code-block:: bash

    $ git clone https://github.com/spoonbobo/sample-aichatbot-riva

The chatbot creation
--------------------

Routes for the web app
**********************

:ref:`apppy` is basically the heart of the web app. :ref:`apppy` defines the routes that decide how users interact with the chatbot when they enter the web app or post a query.

* :code:`dgxData = { "queries": [None], "responses": [dgxWelcomeMsg], "count": 1 }`

    The queries and responses in each chatting session are stored in dgxData object, where count gains increment upon each query sent.

The text input from the chatting page is posted to the Flask backend through :code:`/dgxSubmit`, and the input is redirected to the chatting backend :code:`/dgxChat`. If user input voice message, the audio is posted to the backend through :code:`/dgxUpload`, where the audio is converted to text using DGX ASR service and then redirected to :code:`/dgxChat`.

Each query from user is saved to :file:`dgx_utter.txt` (for re-training the model). NVIDIA DGX Chatbot determines proper response based on className. Lastly, :code:`dgxData` is rendered to :file:`DGXChat.html`  using jinja2 engine, and each of queries and responses is iterated in the chatting session.

Communicate with Riva server
****************************

In :ref:`dgxenquire` gRPC protocol is used to facilitate the communications between the DGX Chatbot and Riva Server. After users record voice messages, the recorded audio is firstly posted from the front-end interface (website) to Flask backend through fetch API, then submitted to Riva Server as :code:`riva_api.riva_asr_pb2.RecognizeRequest`, followed by the Riva Server receiving the request, recognizing the audio, and returning the transcript. 

In :ref:`dgxenquireaudio`, the typed queries or transcripts are submitted to Riva Server as :code:`riva_api.riva_asr_pb2.TextClassRequest`, followed by the Riva Server receiving the request, classifying the queries, and returning the intent classified. Then, the chatbot utters the response to users based on the intent.

Web template
************

A simple HTML template :ref:`dgxchat` is created for displaying chatting interface.

