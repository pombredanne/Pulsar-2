
.. note:: After syncing a new version of a beacon to salt, the salt-minion
    must be restarted to pick up the change. See
    https://github.com/saltstack/salt/issues/35960 for more info

.. _pulsar_introduction:

Introduction
============

Is your infrastructure immutable? Are you sure?

Pulsar is designed to monitor for file system events, acting as a real-time
File Integrity Monitoring (FIM) agent. Pulsar is composed of a custom Salt
beacon that watches for these events and hooks into the returner system for
alerting and reporting.

In other words, you can recieve real-time alerts for unscheduled file system
modifications *anywhere* you want to recieve them.

We've designed Pulsar to be lightweight and not dependent on a Salt Master. It
simply watches for events and directly sends them to one of the Pulsar
returner destinations (see the Quasar_ repository for more on these).

.. _Quasar: https://github.com/hubblestack/quasar

Two different installation methods are outlined below. The first method is more
stable (and therefore recommended). This method uses Salt's package manager to
track versioned, packaged updates to Hubble's components.

The second method installs directly from git. It should be considered bleeding
edge and possibly unstable.

.. _pulsar_installation:

Installation
============

Each of the four HubbleStack components have been packaged for use with Salt's
Package Manager (SPM). Note that all SPM installation commands should be done
on the *Salt Master*.

.. _pulsar_installation_required_configuration:

Required Configuration
----------------------

Salt's Package Manager (SPM) installs files into ``/srv/spm/{salt,pillar}``.
Ensure that this path is defined in your Salt Master's ``file_roots``:

.. code-block:: yaml

    file_roots:
      - /srv/salt
      - /srv/spm/salt

.. note:: This should be the default value. To verify, run: ``salt-call config.get file_roots``

.. tip:: Remember to restart the Salt Master after making any change to the configuration.

.. _pulsar_installation_required_packages:

Required Packages
-----------------

There is a hard requirement on the ``pyinotify`` Python library for each minion
that will run the Pulsar FIM beacon.

.. _pulsar_installation_rhel:

Red Hat / CentOS
~~~~~~~~~~~~~~~~

.. code-block:: shell

    salt \* pkg.install python-inotify

.. _pulsar_installation_deb:

Debian / Ubuntu
~~~~~~~~~~~~~~~

.. code-block:: shell

    salt \* pkg.install python-pyinotify

.. _pulsar_installation_packages:

Installation (Packages)
-----------------------

Installation is as easy as downloading and installing a package. (Note: in
future releases you'll be able to subscribe directly to our HubbleStack SPM
repo for updates and bugfixes!)

.. code-block:: shell

    wget http://spm.hubblestack.io/pulsar/hubblestack_pulsar-2016.9.4-1.spm
    spm local install hubblestack_pulsar-2016.9.4-1.spm

You should now be able to sync the new modules to your minion(s) using the
``sync_modules`` Salt utility:

.. code-block:: shell

    salt \* saltutil.sync_beacons

Copy the ``pillar.example`` into your Salt pillar, renaming is as desired
(perhaps ``hubblestack_pulsar.sls``) and target it to selected minions.

.. code-block:: shell

    base:
      '*':
        - hubblestack_pulsar

.. code-block:: shell

    salt \* saltutil.refresh_pillar

Once these modules are synced you are ready to begin running the Pulsar beacon.

Skip to :ref:`Usage <pulsar_usage>`.

.. _pulsar_installation_manual:

Installation (Manual)
---------------------

Place ``_beacons/pulsar.py`` into your ``_beacons/`` directory, and sync it to
the minions.

.. code-block:: shell

    git clone https://github.com/hubblestack/pulsar.git hubblestack-pulsar.git
    cd hubblestack-pulsar.git
    mkdir -p /srv/salt/_beacons/
    cp _beacons/pulsar.py /srv/salt/_beacons/
    mkdir /srv/salt/hubblestack_pulsar
    cp hubblestack_pulsar/hubblestack_pulsar_config.yaml /srv/salt/hubblestack_pulsar
    cp hubblestack_pulsar/pillar.example /srv/pillar/hubblestack_pulsar.sls
    salt \* saltutil.sync_beacons

Target the copied ``hubblestack_pulsar.sls`` to selected minions.

.. code-block:: shell

    base:
      '*':
        - hubblestack_pulsar

.. code-block:: shell

    salt \* saltutil.refresh_pillar

.. _pulsar_usage:

Usage
=====

Once Pulsar is fully running there isn't anything you need to do to interact
with it. It simply runs quietly in the background and sends you alerts.

.. _pulsar_configuration:

Configuration
=============

The default Pulsar configuration (found in ``<pillar.example>``)
is meant to act as a template. It works in tandem with the
``<hubblestack_pulsar_config.yaml>`` file. Every environment will have
different needs and requirements, and we understand that, so we've designed
Pulsar to be flexible.

** pillar.example **

.. code-block:: yaml

    beacons:
      pulsar:
        paths:
          - /var/cache/salt/minion/files/base/hubblestack_pulsar/hubblestack_pulsar_config.yaml
    schedule:
      cache_pulsar:
        function: cp.cache_file
        seconds: 86400
        args:
          - salt://hubblestack_pulsar/hubblestack_pulsar_config.yaml
        return_job: False

** hubblestack_pulsar_config **

.. code-block:: yaml

    /etc: { recurse: True, auto_add: True }
    /bin: { recurse: True, auto_add: True }
    /sbin: { recurse: True, auto_add: True }
    /boot: { recurse: True, auto_add: True }
    /usr/bin: { recurse: True, auto_add: True }
    /usr/sbin: { recurse: True, auto_add: True }
    /usr/local/bin: { recurse: True, auto_add: True }
    /usr/local/sbin: { recurse: True, auto_add: True }
    return: slack_pulsar
    checksum: sha256
    stats: True
    batch: False

In order to receive Pulsar notifications you'll need to install the custom
returners found in the Quasar_ repository.

Example of using the Slack Pulsar returner to recieve FIM notifications:

.. code-block:: yaml

    slack_pulsar:
      as_user: true
      username: calculon
      channel: hubble_pulsar
      api_key: xoxo-xxxxxxxxxxx-xxxxxxxxxxxxxxxxxxxxxxxx

.. tip:: If you need to create a Slack bot, see: https://my.slack.com/services/new/bot

.. _pulsar_configuration_excluding_paths:

Excluding Paths
---------------

There may be certain paths that you want to exclude from this real-time
FIM tool. This can be done using the ``exclude:`` keyword beneath any
defined path.

.. code-block:: yaml

    /var:
      recurse: True
      auto_add: True
      exclude:
        - /var/log
        - /var/spool
        - /var/cache
        - /var/lock

.. _pulsar_under_the_hood:

Under The Hood
==============

Pulsar is written as a Salt beacon, which requires the ``salt-minion`` daemon
to be running. This then acts as an agent that watches for file system events
using Linux's ``inotify`` subsystem.

.. _pulsar_development:

Development
===========

If you're interested in contributing to this project this section outlines the
structure and requirements for Pulsar agent module development.

.. _pulsar_anatomy_of_a_pulsar_module:

Anatomy of a Pulsar module
--------------------------

.. code-block:: python

    # -*- encoding: utf-8 -*-
    '''
    Pulsar agent

    :maintainer: HubbleStack / owner
    :maturity: 20160804
    :platform: Linux
    :requires: SaltStack

    '''
    from __future__ import absolute_import
    import logging

All Pulsar agents should include the above header, expanding the docstring to
include full documentation

Any Pulsar agent should be written as a beacon and send its return data
directly to the Quasar_ endpoint(s). No communication with the master is
required.

.. _pulsar_contribute:

Contribute
==========

If you are interested in contributing or offering feedback to this project feel
free to submit an issue or a pull request. We're very open to community
contribution.
