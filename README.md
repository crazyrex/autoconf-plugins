autoconf-plugins
================

This repository contains libraries for nagios (or other monitoring system) plugins that are self-documented and allow for auto-configuration.
The aim is to ease integration of new plugins and/or new monitoring point, by proposing to the user a full set of ready-to-use plugin configurations.

For example, if the use choose to monitor the file systems of a newcoming server, the system will be able to propose a full list of filesystem present, and the whole configuration could be done is a few clicks.

Plugins specifications
======================

To achieve this the plugins (whatever the langage they use) should follow these requirements :

Auto-documentation
------------------

When called with the `--autodoc` parameter, the plugin **must** return the following informations :
* Logical name (eg. `CHECK-FEATURE`)
* Version (eg. `1.4`)
* Short plugin description (one line only)
* List of input arguments
  * Logical name (eg. `HOST-IP`)
  * Shortcut (eg. `-H`)
  * Short description
  * Format (string | IP | number | percent | boolean | !regexp!)
  * Mandatory ( states if this argument is mandatory : true/false )
  * Discoverable (states if this argument may be guess in the *autoconf* phase : true/false)
  * UsedForDiscovery (states if this argument is mandatory for the *autoconf* to work : true/false)

Auto-configuration
------------------

If **at least one argument is discoverable**, then the plugin may be called with the `--autoconf` parameter. It then returns the list of possible values for the discoverable arguments, with an additional logical name for each instance.

Example
-------

### check_fs.pl Plugin : Preparation

The Perl code is very simple : just import the library :

```perl
use Nagios::Autoconf qw(FORMAT_IP FORMAT_NUMBER);

```

Then initialize the `Nagios::Autoconf` object

```perl
my $plugin = Nagios::Autoconf->new( $PLUGIN_NAME, $Version, description => $PLUGIN_DESC );

```

Then the arguments to be used by the plugin must be defined

```perl
$plugin->addArgument('HOST-IP', format=> FORMAT_IP, description=> 'IP address of host (mandatory)');
$plugin->addArgument('FILESYSTEM', shortcut=> 'n'
			, format=> '/([\w\-\.]+/)*[\w\-\.]*|[A-Z]:'
			, description=> 'Name of the filesystem (mandatory)'
			, discoverable=> 1);
$plugin->addArgument('warning', mandatory=>0, description=> 'warning threshold');
$plugin->addArgument('critical', mandatory=>0, description=> 'critical threshold');

```

Finally, tell the script to read the arguments passed in the command-line. This will also automatically manage the following parameters : `--version`, `--help`, `--autodoc` and `--autoconf`.

```perl
$plugin->processArguments();

```

### check_fs.pl Plugin : Auto-documentation

Once the script has been configuration as stated above, it automatically answers to `--version` or `--help` standard arguments. That way, you are *certain* that the given documentation exactly matches the way the script is supposed to work :

```
$> perl ./perl/check_test.pl --help
CHECK-FS : Version 1.3
Check filesystem (multi-OS : Linux/Windows)
Usage : ./perl/check_test.pl -H <HOST-IP> -n <FILESYSTEM> [ -w <warning> ] [ -c <critical> ]
   -H <HOST-IP> : IP address of host (mandatory)
   -n <FILESYSTEM> : Name of the filesystem (mandatory)
   -w <warning> : warning threshold
   -c <critical> : critical threshold

```

But there's more : With the new parameter `--autodoc`, it is possible to extract the plugin protocol in CSV format, and build a HMI to help user to configure the monitoring instances. (that's the whole point of this library)

```
$> ./check_fs.pl --autodoc
# Autodoc : format CSV
# Plugin :
# Name;Version;Description

CHECK-FS;1.3;Check filesystem (multi-OS : Linux/Windows)

# Arguments :
# Name;Shortcut;Format;Mandatory;Discoverable;UsedForDiscovery;Description

HOST-IP;H;\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3};1;0;1;IP address of host (mandatory)
FILESYSTEM;n;/([\w\-\.]+/)*[\w\-\.]*|[A-Z]:;1;1;0;Name of the filesystem (mandatory)
warning;w;\d+;0;0;0;warning threshold
critical;c;\d+;0;0;0;critical threshold

```

### check_fs.pl Plugin : Auto-configuration

**The `--autoconf` parameter is even more magical.** When you think about it, some nagios plugin would be able to "guess" some of the parameters they can take. For example, a "Check Filesystem" plugin is able to very easily extract the list of available filesystems on the host they're testing.
Such plugin will thus be able to return a list of "proposed" instances, which the user could later on validate, of select which one of these instance they want to keep :
```
$> ./check_fs.pl -H 192.168.1.2 -w 60 -c 75 --autoconf
# Autoconf : format CSV

# INSTANCE_NAME;HOST-IP;FILESYSTEM;warning;critical
FS-ROOT;192.168.1.2;/;60;75
FS-HOME;192.168.1.2;/home;60;75
FS-TEMP;192.168.1.2;/tmp;90;95
FS-DEV;192.168.1.2;/dev;60;75

```

The Plugin may also directly generate ready to use command lines with an additional `--commandline` argument :

```
$> ./check_fs.pl -H 192.168.1.2 -w 60 -c 75 --autoconf --commandline
# Autoconf : command lines
./perl/check_test.pl -H 192.168.1.2 -n / -w 60 -c 75
./perl/check_test.pl -H 192.168.1.2 -n /home -w 60 -c 75
./perl/check_test.pl -H 192.168.1.2 -n /tmp -w 90 -c 95
./perl/check_test.pl -H 192.168.1.2 -n /dev -w 60 -c 75

```

This makes it very easy to test a new plugin or a new server.

In the above examples, some arguments (warning and critical) were given to the command line to fill in some missing values. This is not mandatory too, but very helpful if you want to generate many instance with more or less the same conditions.

# General steps of automatic instance definitions

Given these properties for plugins, here are the steps necessary for automatic import :

1. User choose plugin + poller
2. System calls plugin with `--autodoc`
3. User fills in the "used for discovery" fields
4. System calls plugin with `--autoconf`
5. The user is given a list of instances, and may review/modify it before validation
