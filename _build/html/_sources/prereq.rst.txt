Prerequisites
=============

This section contains the prerequisites for NVIDIA SA Virtual Assistant.

.. note::

    1. Only Linux environment is supported for the instructions below.
    2. You have access to a Volta, Turing, or an NVIDIA Ampere architecture-based GPU.

1. Docker
*********

Docker is required for using containers in NVIDIA GPU Cloud (NGC), install `here <https://docs.docker.com/engine/install/ubuntu/>`_ if you have not.

NVIDIA Contianer Toolkit
------------------------
To enable GPUs in Docker containers, follow `installation guide <https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html>`_ to set up NVIDIA Container Toolkit.


2. NVIDIA AI Software
*********************

NVIDIA GPU Cloud API
--------------------
Register `NGC Key  <https://ngc.nvidia.com/>`_ and make sure you are able to pull containers from NGC through CLI.

NVIDIA TAO Toolkit
------------------
TAO is used to train conversational AI models to be deployed in Riva Server. Follow `TAO Toolkit Quick Start Guide <https://docs.nvidia.com/tao/tao-toolkit/text/tao_toolkit_quick_start_guide.html>`_ to setup TAO runtime.

NVIDIA Riva Skills Quick Start
------------------------------
Download Riva quick start scripts via the command-line with the NGC CLI tool:

.. code:: bash

    $ ngc registry resource download-version "nvidia/riva/riva_quickstart:1.10.0-beta"

NVIDIA Riva ServiceMaker
------------------------
Riva ServiceMaker provides handy tools to deploy trained models for Riva AI Services. Pull the container in CLI:

.. code:: bash

    $ docker pull nvcr.io/nvidia/riva/riva-speech:1.10.0-beta-servicemaker

