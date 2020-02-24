.. _map.jinja:

``map.jinja``: gather formula configuration values
==================================================

The `documentation <https://docs.saltstack.com/en/latest/topics/development/conventions/formulas.html#writing-formulas>`_ gives a quick view of the use of a ``map.jinja`` to gather parameters values for a formula. 

The today best practice is to let ``map.jinja`` handle parameters from all sources and avoid the call of pillars, grains or configuration from ``sls`` files and template directly.


.. contents:: **Table of Contents**


How to set configuration values of a formula
--------------------------------------------

The ``map.jinja`` use several sources where to lookup parameter values:

- YAML files located in the `fileserver <https://docs.saltstack.com/en/latest/ref/configuration/master.html#std:conf_master-fileserver_backend>`_
- configuration collected by `salt['config.get'] <https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.config.html#salt.modules.config.get>`_

The `pillars <https://docs.saltstack.com/en/latest/topics/pillar/>`_ are rendered by the SaltStack master and could became costly with lots of minions with many pillar values.

As a good practice, you should:

- store non secret data in YAML files distributed by the `fileserver <https://docs.saltstack.com/en/latest/ref/configuration/master.html#std:conf_master-fileserver_backend>`_
- store sensible data
  - in pillar (and look for the use of something like `pillar.vault <https://docs.saltstack.com/en/latest/ref/pillar/all/salt.pillar.vault.html>`_)
  - in `SDB <https://docs.saltstack.com/en/latest/topics/sdb/index.html>`_ (and look for the use of something like `sbd.vault <https://docs.saltstack.com/en/latest/ref/sdb/all/salt.sdb.vault.html>`_)


Configuration from YAML files
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Each formula comes with one or more YAML files, all located under the ``<formula-name>/parameters`` directory. The parameter values are initialised with the mandatory ``defaults.yaml``. It should contain sane default values for the formula.

Then, ``map.jinja`` will load configuration using `salt['config.get'] <https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.config.html#salt.modules.config.get>`_ from a configurable list, by default, this list is the following:

#. ``osarch``: the CPU architecture of the minion
#. ``os_family``: the family of the operating system (e.g. ``Debian`` for an ``Ubuntu``)
#. ``os``: the name of the operating system (e.g. ``Ubuntu``)
#. ``osfinger``: the concatenation of the operating system name and it's version string (e.g. ``Debian-10``)
#. ``id``: the ``ID`` of the minion

After the configuration values are initialised with ``defaults.yaml``, for each configuration of the list, in the order, ``map.jinja`` will:

#. lookup the value of the configuration
#. load the values of ``parameters/<config>/<config value>.yaml``
#. merge the loaded values with the previous ones using `salt.slsutil.merge <https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.slsutil.html>`_

Each YAML parameter file:

- is optional, there will be no error if the file does not exists
- when it exists
  - the configuration values must be under the top level ``values`` key
  - the merging strategy of ``salt.slsutil.merge`` can be configured by the top level ``strategy`` key, for example ``strategy: 'recurse'``, the default is ``smart``
  - the merging of list for the ``recurse`` and ``overwrite`` merging strategy can be configured with the top level key ``merge_lists``, for example ``merge_lists: 'true'``

If the ``config.get`` lookup failed, the configuration name will be used literally as a custom path to a YAML file, for example: ``any/path/can/be/used/here.yaml`` will result in the loading of ``parameters/any/path/can/be/used/here.yaml``.

You can override the list of configuration to lookup by setting ``map_jinja:sources`` from 3 places, in the lookup order:

#. ``defaults.yaml`` for the author of the formula
#. pillar root (or anywhere reachable by ``salt['config.get']``
#. under ``<formula>:map_jinja:sources``

Configuration from ``salt['config.get']``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

After the configuration is loaded from YAML files, ``map.jinja`` lookup the ``<formula name>`` with `salt['config.get'] <https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.config.html#salt.modules.config.get>`_ and then merge with the previously loaded values:

- first the ``<formula>:lookup`` dict
- then the complete ``<formula>`` dict

Global view of the order of preferences
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To make things clear, here is a complete example of the load order of formula configuration values for an ``AMD64`` ``Ubuntu 18.04`` minion named ``minion1.example.net``:

#. ``parameters/defaults.yaml``
#. ``parameters/osarch/amd64.yaml``
#. ``parameters/os_family/Debian.yaml``
#. ``parameters/os/Ubunta.yaml``
#. ``parameters/osfinger/Ubunta-18.04.yaml``
#. ``parameters/id/minion1.example.net``
#. ``salt['config.get']('<formula>:lookup')``
#. ``salt['config.get']('<formula>')``


Use formula configuration values in `sls`
-----------------------------------------

The good practice for ``map.jinja`` are:

- to be located at the root of the formula named directory (e.g. ``mysql-formula/mysql/map.jinja``)
- to export a variable with the same name than the formula itself. As an example, for ``mysql-formula`` it will be ``mysql``.

Here is the best way to use it in an ``sls`` file:

.. code-block:: sls

    {#- Get the `tplroot` from `tpldir` #}
    {%- set tplroot = tpldir.split('/')[0] %}
    {%- from tplroot | path_join('map.jinja') import TEMPLATE with context %}

    test-does-nothing-but-display-TEMPLATE-as-json:
      test.nop:
        - name: {{ TEMPLATE | json }}



Use formula configuration values in templates
---------------------------------------------

When you need to process salt templates, you should avoid calling ``salt['config.get']`` (or ``salt['pillar.get']`` and ``salt['grains.get']``) directly from the template. All the needed values should be available within the variable exported by ``map.jinja``.

Here is an example based on ``template-formula/TEMPLATE/config/file.sls``

.. code-block:: sls

    # -*- coding: utf-8 -*-
    # vim: ft=sls
    
    {#- Get the `tplroot` from `tpldir` #}
    {%- set tplroot = tpldir.split('/')[0] %}
    {%- set sls_package_install = tplroot ~ '.package.install' %}
    {%- from tplroot ~ "/map.jinja" import TEMPLATE with context %}
    {%- from tplroot ~ "/libtofs.jinja" import files_switch with context %}
    
    include:
      - {{ sls_package_install }}
    
    TEMPLATE-config-file-file-managed:
      file.managed:
        - name: {{ TEMPLATE.config }}
        - source: {{ files_switch(['example.tmpl'],
                                  lookup='TEMPLATE-config-file-file-managed'
                     )
                  }}
        - mode: 644
        - user: root
        - group: {{ TEMPLATE.rootgroup }}
        - makedirs: True
        - template: jinja
        - require:
          - sls: {{ sls_package_install }}
        - context:
            TEMPLATE: {{ TEMPLATE | json }}

This ``sls`` file expose a ``TEMPLATE`` context variable to the jinja template which could be used like this:

.. code-block:: jinja

    ########################################################################
    # File managed by Salt at <{{ source }}>.
    # Your changes will be overwritten.
    ########################################################################
    
    This is another example file from SaltStack template-formula.
    
    # This is here for testing purposes
    {{ TEMPLATE | json }}

    winner of the merge: {{ TEMPLATE['winner'] }}
