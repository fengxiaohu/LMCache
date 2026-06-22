Extending the CLI
=================

This guide explains how to add new subcommands to the ``lmcache`` CLI.

Architecture Overview
---------------------

The CLI uses plugin-style auto-discovery instead of manual command
registration:

1. Each command is a class inheriting from ``BaseCommand``.
2. Top-level command modules live under ``lmcache/cli/commands/``.
3. Command groups inherit from ``CompositeCommand`` and discover child
   commands from their own package.
4. At startup, discovered commands are instantiated into ``ALL_COMMANDS`` and
   registered with argparse.

``BaseCommand`` is an abstract class with a small set of required methods:
``name()``, ``help()``, ``add_arguments()``, and ``execute()``. Forgetting any
of them raises ``TypeError`` at instantiation time.

For the complete extension walkthrough, including nested command groups, see
:doc:`/extension/cli`.

Adding a Top-Level Command
--------------------------

Create one new module under ``lmcache/cli/commands/`` with a concrete
``BaseCommand`` subclass:

.. code-block:: python

   # SPDX-License-Identifier: Apache-2.0
   import argparse

   from lmcache.cli.commands.base import BaseCommand


   class DescribeCommand(BaseCommand):
       """Describe a running KV cache server."""

       def name(self) -> str:
           return "describe"

       def help(self) -> str:
           return "Describe a running KV cache server."

       def add_arguments(self, parser: argparse.ArgumentParser) -> None:
           parser.add_argument(
               "--url",
               required=True,
               help="LMCache HTTP server URL (e.g. http://localhost:8000)",
           )

       def execute(self, args: argparse.Namespace) -> None:
           metrics = self.create_metrics("Describe KV Cache", args)
           metrics.add("status", "Status", "OK")
           metrics.add("chunks", "Cached chunks", 1024)
           metrics.emit()

No import or registry edit is required. ``lmcache/cli/commands/__init__.py``
uses ``discover_subclasses`` to find concrete ``BaseCommand`` subclasses in
direct child modules and packages.

Adding a Nested Command
-----------------------

To add a command under an existing ``CompositeCommand`` group, add a concrete
``BaseCommand`` subclass as a sibling module inside that group's package. For
example, ``lmcache tool list-commands`` lives under ``lmcache/cli/commands/tool/``
and is discovered by ``ToolCommand`` without editing ``tool/__init__.py``.

Use modules prefixed with ``_`` for helpers, such as ``_helpers.py``. They are
excluded from auto-discovery.

Inspecting Discovered Commands
------------------------------

Run the built-in command tree inspector to see what the auto-discovery path
registered:

.. code-block:: bash

   lmcache tool list-commands
   lmcache tool list-commands --format json

Using the Metrics System
------------------------

The metrics system uses a **handler + formatter** architecture:

- **Metrics** - the collector. Holds sections and entries.
- **Handler** - the destination (stdout, file, etc.).
- **Formatter** - the rendering (ASCII table, JSON, etc.).

``BaseCommand.create_metrics()`` sets up default handlers automatically, so
command authors just build metrics and call ``emit()``:

.. code-block:: python

   def execute(self, args: argparse.Namespace) -> None:
       metrics = self.create_metrics("Bench KV Cache Result", args)

       metrics.add_section("ops", "Operations (ops/s)")
       metrics["ops"].add("store", "Store", 41.3)
       metrics["ops"].add("retrieve", "Retrieve", 127.3)

       metrics.add("status", "Status", "OK")
       metrics.emit()

The ``--format``, ``--output``, and ``--quiet`` flags are added automatically by
``BaseCommand.register()``. Subcommands do not need to add them manually.
