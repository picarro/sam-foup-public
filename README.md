Getting Started
===============

Contents
--------

This repository contains Picarro software components that are shared with collaborators as part of the SAM FOUP project. This includes:

* [proto](proto): gRPC/ProtoBuf Interface Definition files.
* [releases](releases): Installable Debian package, `picarro-sam`.


### Package Contents

The `picarro-sam` package includes the main gRPC server application, `picarro-edge`. Edge provides the following services, with corresponding interface definitions in the [proto](proto) folder:

* [picarro.sam.foup.FOUP](proto/foup.proto) to control and monitor FOUP measurements
* [picarro.sam.controller.Controller](proto/controller.proto) to monitor events from the SAM controller, notably health alerts.

The package also includes command-line utilities and Python modules to interact with these services, described below.


Runtime Environment
-------------------

Installing and running the Edge server as well as the command-line tools requires a recent version of Linux ([Debian 12](https://www.debian.org/distrib/), [Ubuntu 24.04](https://ubuntu.com/download/desktop), or newer).

Python client modules are also included and can be run from any OS with Python 3.9 or newer installed; see the section **Python Package ("wheel")** below.


Installing the `picarro-sam` Debian package
-------------------------------------------

To install the release on your sysem, open a Terminal window and type the following command (replace `VERSION` as appropriate):

  ```shell
  sudo dpkg -i releases/picarro-sam-VERSION.deb
  ```

(The `sudo` command is used to gain `root` privileges in order to run the `dpkg` utility).

The first time you run this command it may fail due to missing runtime dependencies.  In this case, try again using the following command to automatically retrieve and install those:

  ```shell
  sudo apt -y install ./releases/picarro-sam-VERSION.deb
  ```

(Note that the leading `./` is required for the `apt` tool to recognize this as a local filename, rather than a package name to be retrieved from your Debian/Ubuntu repository).


Launching Picarro Edge
----------------------

By default you do not need to do anything, as `picarro-edge` is installed and started in simulation mode as a system service.

If you prefer to run it interactively, for instance to see log messages printed to your terminal, run the following:

  ```shell
  sudo systemctl stop picarro-edge      # Stop the service
  sudo systemctl disable picarro-edge   # Keep it from automatically launching at boot
  /usr/sbin/picarro-edge                # Launch Edge in the foreground
  ```

If you later wish to re-enable the system service, use the following:

  ```shell
  sudo systemctl enable picarro-edge
  sudo systemctl start picarro-edge
  ```

#### A note on simulation

By default, Edge runs in simulation mode, with the following caveats:
 * **FOUP** control functions are available, and will trigger corresponding `op_status`, `job_status`, and `job_result` messages as the run progresses
 * **Controller** observation is present but not very useful, as no health alerts or other events are generated
 * **CRDS Data** functions are not simulated



Interacting with the server
---------------------------

### Command Line Tools

The following command-line tools are provided to interact with the corresponding gRPC services:

* `foup-api-tool` to control and monitor measurements via the `FOUP` service
* `crds-api-tool` to retrieve detailed CRDS measurement data (not available in simulation mode)
* `controller-api-tool` to monitor SAM Core events including health alerts (no events in simulation mode)

Use the `--help` option for any of these commands for detailed usage, or `--help commands` for a shorter synopsis of available subcommands.


#### Basic connectivity check

To check basic client/server connectivity using the provided `foup-api-tool`, use

  ```shell
  foup-api-tool get_info
  ```

This will report the API versions used by the test client and the server, as well as the server name (executable name).


#### Monitoring FOUP Events

To start a passive listener for events from the **FOUP** service (see the `Signal` message in [foup.proto](proto/foup.proto)), run

  ```shell
  foup-api-tool monitor
  ```

This will print out any events that it receives back to your terminal.  Press ENTER to stop and exit the program.

**Tip:** Leave this command running in its own terminal window, while you proceed to the next step.


#### Monitoring Instrument Health Events

Similarly, to start a passive listener for `analyzer_health` and `analyzer_driver` events from the **Controller** service (see the `Signal` message in [controller.proto](proto/controller.proto)), use:

  ```shell
  controller-api-tool monitor analyzer_health analyzer_driver
  ```

  (You can omit the `analyzer_health` and `analyzer_driver` keywords to stream additional events from the Controller service, though this will be noisy).


#### Starting a FOUP measurement

Launching a FOUP job requires three arguments:

* The action type, e.g. `MEASURE_1` or `MEASURE_2`

* A (presumably unique) FOUP ID

* A measurement duration, specified in seconds.


For example, the following will perform a 90-second measurement:

  ```shell
  foup-api-tool start_job MEASURE_1 "My First Foup" 90
  ```

A unique Run ID will be printed on the terminal, which you can later use to abort the run or obtain results.

If you are monitoring FOUP events as per above you will see the job progress underway, and concentration values once the job ends.


#### Aborting a FOUP measurement

While underway, a measurement can be cancelled using

  ```shell
  foup-api-tool abort_job RUN_ID
  ```

where `RUN_ID` is the ID that was returned from the `start_job` command above.

#### Obtaining job results

To obtain results from a previously completed job, use

  ```shell
  foup-api-tool get_results RUN_ID
  ```

where `RUN_ID` is the ID that was returned from the `start_job` command above.


### Python client modules

#### Client module

Various Python gRPC client modules are installed under the standard Python 3.x module folder, `/usr/lib/python3/dist-packages`. These contain wrapper methods for all of the above service functions. Specifically:

*  `picarro/sam/foup/grpc/client.py`: control and monitor measurements via the `FOUP` service
*  `picarro/sam/controller/grpc/client.py`: monitor SAM Core events including health alerts

These modules require that the `python3-grpcio` and `python3-protobuf`packages be installed on the system.  To do so, run the following:

  ```bash
  sudo apt install python3-grpcio python3-protobuf
  ```

#### SAM Shell

An interative Python shell with preloaded gRPC modules can be launched using the `samshell` command:

  ```shell
  samshell
  ```

You should see a brief legend with an interative Python prompt:

   ```shell

      Interactive Service Control.  Subsystems loaded:

          crds       - CRDS Data gRPC client
          crds_rest  - CRDS Data REST client
          foup       - FOUP gRPC client
          controller - Controller client

      ProtoBuf types are generally loaded into namespaces matching the package
      names from their respective `.proto` files:

          google.protobuf    - Well-known types from Google
          picarro.*          - Picarro modules
          picarro.sam.*      - Custom types specific to SAM
          picarro.protobuf.* - General utilities and wrapper modules

      Use 'help(subsystem)' to list available methods

  >>>
  ```

From here, you will see a number of service methods and associated ProtoBuf data types - for instance, a top-level `foup` instance of the `FOUP.Client()` class above.

* To monitor service events, use `foup.start_notify_signals(CALLBACK)`.
  For instance, to print events to your terminal, use:

  ```python
  >>> foup.start_notify_signals(protobuf.utils.print_messsage)
  ```

* To start a job, use `foup.start_job()`:

  ```python
  >>> foup.start_job(action=picarro.sam.foup.ACTION_MEASURE_1, foup_id="My Foup", duration=90)
  ```


**Tip**: Use Pythons's interactive command line completion to list available completions at various points.  For instance, once you have typed `foup.start_job(action=picarro.sam.foup.ACTION_`, press **[TAB]** twice to show a list of possible completions starting with `ACTION_`.

**Tip**: Use Python's `help()` function for documentation and calling syntax.  For instance, use `help(foup)` to get a list of available methods in the `foup` instance.


### Python package ("Wheel")

The Python client modules are also available in an installable [Python wheel](https://pythonwheels.com/), which allows them to be used in a Python 3.9 or higher environment on any platform. 

After you download the file `picarro_sam-VERSION-py3-none-any.whl` from the [releases](releases) folder onto your system, you have two options.

#### Install the wheel with PIP

You can use the [Package Installer for Python](https://pypi.org/project/pip/) (PIP) to install this wheel, preferably into a [Python Virtual Environment](https://virtualenv.pypa.io/en/latest/user_guide.html).  The steps vary slighly based on your host OS; on a Debian-based system, follow these steps:

1. Install required Python modules:

   ```bash
   sudo apt install python3-virtualenv python3-pip
   ```

2. Create a virtual environment.  For this example we'll do so in `$HOME/picarro`:

   ```bash
   virtualenv $HOME/picarro
   ```

3. "Activate" this environment. This effectively adjusts your `$PATH` environment variable so that commands like `python` and `pip` are launched from within this environment instead of your system folders:

   ```bash
   $HOME/picarro/bin/activate
   ```

   You will need to repeat this command whenever you want to use this environment from a new shell (i.e. Terminal window).

4. Install the wheel within this newly activated environment (replacing its path as appropriate):

   ```bash
   pip install $HOME/Downloads/picarro_sam-VERSION-py3-none-any.whl
   ```

5. Launch the `samshell` interactive prompt from within this environment:

   ```bash
   python -i -m picarro.sam.shell
   ```

   If you wish to connect to a Picarro Edge server running on another host, use:

   ```bash
   python -i -m picarro.sam.shell --host ADDRESS
   ```

   (Replace `ADDRESS` with the resolvable name or IP address of the remote host).


#### Launch modules directly from wheel

Alternatively, you can load modules from the wheel without installing it.  To do so, launch your OS native Python interpreter but with the environment variable `PYTHONPATH` pointing to the `.whl` file:

   ```bash
   PYTHONPATH=$HOME/Downloads/picarro_sam-VERSION-py3-none-any.whl \
      python3 -i -m picarro.sam.shell
   ```

### Web UI

Picarro Edge is built with support for [gRPC reflection](https://grpc.io/docs/guides/reflection/), and is therefore accesible with tools such as [gRPC-UI](https://www.fullstory.com/blog/grpcui-dont-grpc-without-it/).

Follow the instructions the [gRPC-UI GitHub page](https://github.com/fullstorydev/grpcui) to install this tool, then launch it as follows:

  ```bash
  $HOME/go/bin/grpcui -plaintext localhost:3343
  ```

This will bring an interactive gRPC request builder in your default web browser.

**Caveat:** gRPC-UI does not seem to handle continuous server streams, so is not a suitable tool for the `watch()` method.


#### Example: Start a FOUP measurement

To launch a new measurement:

* From the drop-down list near the top, select `picarro.sam.foup.FOUP`
* Among the available methods just under this service name, choose `start_job`
* Check `action [ ]` and then choose `ACTION_MEAUSURE_1`
* Check `foup_id [ ]` and type in a unique FOUP ID
* Check `duration [ ]` and select a duration, e.g. 1 minute.



Questions? Comments? Concerns?
------------------------------

Please reach out to me.

>  Tor Slettnes  
>  tslettnes@picarro.com  
>  +1-925-8888-TOR
