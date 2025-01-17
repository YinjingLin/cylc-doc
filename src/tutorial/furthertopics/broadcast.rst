Broadcast
=========

This tutorial walks you through using ``cylc broadcast`` which can be used
to change :ref:`task runtime configuration <tutorial-runtime>` in a
running workflow, on-the-fly.


Purpose
-------

``cylc broadcast`` can be used to change any ``[runtime]`` setting whilst the
workflow is running.

The standard use of ``cylc broadcast`` is to update the workflow to an
unexpected change in configuration, for example modifying the host a task
runs on.


Standalone Example
------------------

Create a new workflow in the ``cylc-src`` directory called
``tutorial-broadcast``::

   mkdir ~/cylc-src/tutorial-broadcast
   cd ~/cylc-src/tutorial-broadcast

Copy the following configuration into a :cylc:conf:`flow.cylc` file:

.. code-block:: cylc

   [scheduling]
       initial cycle point = 1012
       [[graph]]
           R1 = wipe_log => announce
           PT1H = announce[-PT1H] => announce

   [runtime]
       [[wipe_log]]
           # Delete any files in the workflow's "share" directory.
           script = rm "${CYLC_WORKFLOW_SHARE_DIR}/knights" || true

       [[announce]]
           script = echo "${CYLC_TASK_CYCLE_POINT} - ${MESSAGE}" >> "${FILE}"
           [[[environment]]]
               WORD = ni
               MESSAGE = We are the knights who say \"${WORD}\"!
               FILE = "${CYLC_WORKFLOW_SHARE_DIR}/knights"

We now have a workflow with an ``announce`` task which runs every hour, writing a
message to a log file (``share/knights``) when it does so. For the first cycle
the log entry will look like this::

   10120101T0000Z - We are the knights who say "ni"!

The ``cylc broadcast`` command enables us to change runtime configuration
whilst the workflow is running. For instance we could change the value of the
``WORD`` environment variable using the command::

   cylc broadcast tutorial-broadcast -n announce -s "[environment]WORD=it"

* ``tutorial-broadcast`` is the name of the workflow.
* ``-n announce`` tells Cylc we want to change the runtime configuration of the
  ``announce`` task.
* ``-s "[environment]WORD=it"`` changes the value of the ``WORD`` environment
  variable to ``it``.

Run the workflow then try using the ``cylc broadcast`` command to change the
message::

   cylc validate .
   cylc install
   cylc play tutorial-broadcast
   cylc broadcast tutorial-broadcast -n announce -s "[environment]WORD=it"

Inspect the ``share/knights`` file, you should see the message change at
certain points.

Stop the workflow::

   cylc stop tutorial-broadcast


In-Situ Example
---------------

We can call ``cylc broadcast`` from within a task's script. This effectively
provides the ability for tasks to communicate between themselves.

It is almost always better for tasks to communicate using files but there are
some niche situations where communicating via ``cylc broadcast`` is justified.
This tutorial walks you through using ``cylc broadcast`` to communicate between
tasks.

.. TODO - examples of this?

Add the following recurrence to the ``graph`` section:

.. code-block:: cylc

           PT3H = announce[-PT1H] => change_word => announce

The ``change_word`` task runs the ``cylc broadcast`` command to randomly
change the ``WORD`` environment variable used by the ``announce`` task.

Add the following runtime configuration to the ``runtime`` section:

.. code-block:: cylc

       [[change_word]]
           script = """
               # Select random word.
               IFS=',' read -r -a WORDS <<< $WORDS
               WORD=${WORDS[$(date +%s) % ${#WORDS[@]}]}

               # Broadcast random word to the announce task.
               cylc broadcast $CYLC_WORKFLOW_ID -n announce -s "[environment]WORD=${WORD}"
           """
           [[[environment]]]
               WORDS = ni, it, ekke ekke ptang zoo boing

Re-install and run the workflow, and inspect the log.
You should see the message change randomly
after every third entry (because the ``change_word`` task runs every 3 hours)
e.g::

   10120101T0000Z - We are the knights who say "ni"!
   10120101T0100Z - We are the knights who say "ni"!
   10120101T0200Z - We are the knights who say "ni"!
   10120101T0300Z - We are the knights who say "ekke ekke ptang zoo boing!"

Stop the workflow::

   cylc stop tutorial-broadcast
