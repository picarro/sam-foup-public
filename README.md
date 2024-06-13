Getting Started
===============

Contents
--------

This repository contains Picarro software components that are shared with Brooks Automation as part of the SAM FOUP project.  This includes:

* [proto](proto): gRPC/ProtoBuf Interface Definition files.
* [releases](releases): Installable Debian package, `picarro-sam`.


### Package Contents

The `picarro-sam` package includes the main gRPC server application, `picarro-edge`. Edge provides the following services, with corresponding interface definitions in the [proto](proto) folder:

* [picarro.sam.foup.FOUP](proto/foup.proto) to control and monitor FOUP measurements
* [picarro.sam.controller.Controller](proto/controller.proto) to monitor events from the SAM controller, notably health alerts.

The package also includes command-line utilities and Python modules to interact with these services, described below.


Runtime Environment
-------------------

Installing and running the software in this repository requires a recent version of Linux ([Debian 12](https://www.debian.org/distrib/), [Ubuntu 24.04](https://ubuntu.com/download/desktop), or newer).


Installing the `picarro-sam` Debian package
-------------------------------------------

To install the release on your sysem, open a Terminal window and type the following command (replace `VERSION` as appropriate):

  ```shell
  $ sudo dpkg -i releases/picarro-sam-VERSION.deb
  ```

(The `sudo` command is used to gain `root` privileges in order to run the `dpkg` utility).

The first time you run this command it may fail due to missing runtime dependencies.  In this case, try again using the following command to automatically retrieve and install those:

  ```shell
  $ sudo apt -y install ./releases/picarro-sam-VERSION.deb
  ```

(Note that the leading `./` is required for the `apt` tool to recognize this as a local filename, rather than a package name to be retrieved from your Debian/Ubuntu repository).


Launching Picarro Edge
----------------------

By default you do not need to do anything, as `picarro-edge` is installed and started in simulation mode as a system service.

If you prefer to run it interactively, for instance to see log messages printed to your terminal, run the following:

  ```shell
  $ sudo systemctl stop picarro-edge      # Stop the service
  $ sudo systemctl disable picarro-edge   # Keep it from automatically launching at boot
  $ /opt/picarro/sbin/picarro-edge        # Launch Edge in the foreground
  ```

If you later wish to re-enable the system service, use the following:

  ```shell
  $ sudo systemctl enable /opt/picarro/lib/systemd/system/picarro-edge.service
  $ sudo systemctl start picarro-edge
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

* `/opt/picarro/bin/foup-api-tool` to control and monitor measurements via the `FOUP` service
* `/opt/picarro/bin/crds-api-tool` to retrieve detailed CRDS measurement data (not available in simulation mode)
* `/opt/picarro/bin/controller-api-tool` to monitor SAM Core events including health alerts (no events in simulation mode)

Use the `--help` option for any of these commands for detailed usage, or `--help commands` for a shorter synopsis of available subcommands.


#### Basic connectivity check

To check basic client/server connectivity using the provided `foup-api-tool`, use

    ```shell
    $ /opt/picarro/bin/foup-api-tool get_info
    ```

This will report the API versions used by the test client and the server, as well as the server name (executable name).


#### Monitoring FOUP Events

To start a passive listener for events from the **FOUP** service (see the `Signal` message in [foup.proto](proto/foup.proto)), run

    ```shell
    $ /opt/picarro/bin/foup-api-tool monitor
    ```

This will print out any events that it receives back to your terminal.  Press ENTER to stop and exit the program.

**Tip:** Leave this command running in its own terminal window, while you proceed to the next step.


#### Monitoring Instrument Health Events

Similarly, to start a passive listener for events from the **Controller** service (see the `Signal` message in [controller.proto](proto/controller.proto)), use:

  ```shell
  $ /opt/picarro/bin/controller-api-tool monitor analyzer_health
  ```

  (You can omit the `analyzer_health` keyword to stream additional events from the Controller service, though this will be noisy).


#### Starting a FOUP measurement

Launching a FOUP job requires three arguments:

* The action type, e.g. `MEASURE_1` or `MEASURE_2`

* A (presumably unique) FOUP ID

* A measurement duration, specified in seconds.


For example, the following will perform a 90-second measurement:

    ```shell
    $ /opt/picarro/bin/foup-api-tool start_job MEASURE_1 "My First Foup" 90
    ```

A unique Run ID will be printed on the terminal, which you can later use to abort the run or obtain results.

If you are monitoring FOUP events as per above you will see the job progress underway, and concentration values once the job ends.


#### Aborting a FOUP measurement

While underway, a measurement can be cancelled using

    ```shell
    $ /opt/picarro/bin/foup-api-tool abort_job RUN_ID
    ```

where `RUN_ID` is the ID that was returned from the `start_job` command above.

#### Obtaining job results

To obtain results from a previously completed job, use

    ```shell
    $ /opt/picarro/bin/foup-api-tool get_results RUN_ID
    ```

where `RUN_ID` is the ID that was returned from the `start_job` command above.


### Python client modules

**NOTE**: The steps below require that the `python3-grpcio` package has been installed. This means you can run them from the ["Develop"](../../../picarro-shared/docker/develop/), but not from the smaller ["Run"](../../../picarro-shared/docker/run/) (i.e. runtime) image.

#### Client module

Various Python gRPC client modules installed under `/opt/picarro/share/python`. These contain wrapper methods for all of the above service functions. Specifically:

*  `/opt/picarro/share/python/sam/foup/grpc/client.py`: control and monitor measurements via the `FOUP` service
*  `/opt/picarro/share/python/sam/controller/grpc/client.py`: monitor SAM Core events including health alerts

At some point these will  be available separately as an installable [Python wheel](https://pythonwheels.com/). For now, these may be useful as a general reference implementation.


#### SAM Shell

An interative Python shell with preloaded gRPC modules can be launched using the `samshell` command:

  ```shell
  $ /opt/picarro/bin/samshell

      Interactive Service Control.  Subsystems loaded:

          crds       - CRDS Data gRPC client
          crds_rest  - CRDS Data REST client
          foup       - FOUP gRPC client
          controller - Controller client

      ProtoBuf types are generally loaded into namespaces matching the package
      names from their respective `.proto` files:

          google.protobuf - Well-known types from Google
          picarro.*       - Picarro custom types
          picarro.sam.*   - Custom types specific to SAM
          protobuf.*      - General utilities and wrapper modules

      Use 'help(subsystem)' to list available methods

  >>>
  ```

From here, you will see a number of service methods and associated ProtoBuf data types - for instance, top-level `foup` instance of the `Client()` class above.

* To monitor service events, use `foup.start_notify_signals(CALLBACK)`.
  For instance, to print events to your terminal, use:

  ```python
  >>> foup.start_notify_signals(protobuf.utils.print_messsage)
  ```

* To start a job, use `foupc.start_job()`:

  ```python
  >>> foup.start_job(action=picarro.sam.foup.ACTION_MEASURE_1, foup_id="My Foup", duration=90)
  ```


**Tip**: Use Pythons's interactive command line completion to list available completions at various points.  For instance, once you have typed `foup.start_job(action=picarro.sam.foup.ACTION_`, press **[TAB]** twice to show a list of possible completions for the word `ACTION_`.

**Tip**: Use Python's `help()` function for documentation and calling syntax.  For instance, use `help(foup)` to get a list of available methods in the `foup` instance.


### Web UI

Picarro Edge is built with support for [gRPC reflection](https://grpc.io/docs/guides/reflection/), and is therefore accesible with tools such as [gRPC-UI](https://www.fullstory.com/blog/grpcui-dont-grpc-without-it/).

Follow the instructions the [gRPC-UI GitHub page](https://github.com/fullstorydev/grpcui) to install this tool, then launch it as follows:

  ```bash
  $ ~/go/bin/grpcui -plaintext localhost:3343
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
