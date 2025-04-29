+++
date = '2025-04-29T00:06:05+01:00'  
draft = false  
title = 'Speeding up MAAS CLI'  
toc = false  
+++

## MAAS CLI Overview

The MAAS CLI is a command-line interface for managing MAAS, a tool for managing and deploying cloud infrastructure. The CLI can be used to perform various tasks such as creating and managing nodes, configuring networks, and deploying applications. Also, it can be used to perform administrative tasks such as configuring users, the database and MAAS itself when it is executed locally on a MAAS region/rack node.

## The problem

The MAAS CLI is known to be pretty slow, taking usually more than 2 seconds to perform any action. This is a known problem that is affecting MAAS since the very beginning. 
This is due to the fact that the CLI is actually building the entiere CLI parser at every invocation, and the administrative commands (such as `createadmin`, `configure-tls` and others) are django commands, and require django to be setup.

For reference see [https://github.com/canonical/maas/blob/6aa9a0d83909d78fe2166762d1d6905a48973149/src/maascli/cli.py#L316](https://github.com/canonical/maas/blob/6aa9a0d83909d78fe2166762d1d6905a48973149/src/maascli/cli.py#L316)

```
    # Setup and the allowed django commands into the maascli.
    management = get_django_management()
    if management is not None and is_maasserver_available():
        os.environ.setdefault(
            "DJANGO_SETTINGS_MODULE", "maasserver.djangosettings.settings"
        )
        from django import setup as django_setup

        django_setup()
        load_regiond_commands(management, parser)

def load_regiond_commands(management, parser):
    """Load the allowed regiond commands into the MAAS cli."""

    class CanonicalizedCommandManagement(management.ManagementUtility):
        def fetch_command(self, subcommand):
            return super().fetch_command(subcommand.replace("-", "_"))

    canonicalized_management = CanonicalizedCommandManagement()

    for name, app, help_text in REGIOND_COMMANDS:
        klass = management.load_command_class(app, name.replace("-", "_"))
        if help_text is None:
            help_text = klass.help
        command_parser = parser.subparsers.add_parser(
            safe_name(name), help=help_text, description=help_text
        )
        klass.add_arguments(command_parser)
        command_parser.set_defaults(
            execute=partial(run_regiond_command, canonicalized_management)
        )
```

Also, note that `is_maasserver_available` will always return `True` if it's a snap environment. 




## Speeding up the CLI

It does not make much sense to load django also when the user wants to call the "cloud" commands on a remote server. So, I have moved all the django runtime dependencies to the inner commands so to load them only when these commands are actually executed. This way, we can still create the parser without the need to loading django. 

Before

```
$ time maas admin machines read
Success.
Machine-readable output follows:
[]

real 0m2.185s
user 0m1.776s
sys 0m0.163s
```

After

```
$ time maas admin machines read
Success.
Machine-readable output follows:
[]

real 0m0.972s
user 0m0.673s
sys 0m0.080s
```

TL;DR: if you have a project with a CLI that is running both django and non-django commands, you can speed it up by moving the django runtime dependencies to the inner commands.

For the full implementation, see [https://code.launchpad.net/~r00ta/maas/+git/maas/+merge/485194](https://code.launchpad.net/~r00ta/maas/+git/maas/+merge/485194)