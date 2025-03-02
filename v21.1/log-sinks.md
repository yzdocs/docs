---
title: Log Sinks
summary: Reference documentation for log sinks and supported fields.
toc: true
---

The supported log output sink types are documented below.

- [Output to files](#output-to-files)
- [Output to Fluentd-compatible log collectors](#output-to-fluentd-compatible-log-collectors)
- [Standard error stream](#standard-error-stream)


<a name="output-to-files">

## Sink type: Output to files


This sink type causes logging data to be captured into log files in a
configurable logging directory.

The configuration key under the `sinks` key in the YAML
configuration is `file-groups`. Example configuration:

~~~ yaml
sinks:
   file-groups:           # file group configurations start here
      health:             # defines one group called "health"
         channels: HEALTH
~~~

Each generated log file is prefixed by the name of the process,
followed by the name of the group, separated by a hyphen. For example,
the group `health` will generate files named `cockroach-health.XXX.log`,
assuming the process is named `cockroach`. (A user can influence the
prefix by renaming the program executable.)

The files are named so that a lexicographical sort of the
directory contents presents the file in creation order.

A symlink (e.g. `cockroach-health.log`) for each group points to the latest generated log file.

Every new file group sink configured automatically inherits
the configurations set in the `file-defaults` section.

For example:

~~~ yaml
file-defaults:
    redactable: false # default: disable redaction markers
    dir: logs
sinks:
  file-groups:
    health:
       channels: HEALTH
       # This sink has redactable set to false,
       # as the setting is inherited from file-defaults
       # unless overridden here.
       #
       # Example override:
       dir: health-logs # override the default 'logs'
~~~

{{site.data.alerts.callout_success}}
Run `cockroach debug check-log-config` to verify the effect of defaults inheritance.
{{site.data.alerts.end}}



Type-specific configuration options:

| Field | Description |
|--|--|
| `channels` | the list of logging channels that use this sink. See the [channel selection configuration](#channel-format) section for details.  |
| `dir` | specifies the output directory for files generated by this sink. Inherited from `file-defaults.dir` if not specified. |
| `max-file-size` | the approximate maximum size of individual files generated by this sink. Inherited from `file-defaults.max-file-size` if not specified. |
| `max-group-size` | the approximate maximum combined size of all files to be preserved for this sink. An asynchronous garbage collection removes files that cause the file set to grow beyond this specified size. Inherited from `file-defaults.max-group-size` if not specified. |
| `buffered-writes` | specifies whether to flush on every log write. Inherited from `file-defaults.buffered-writes` if not specified. |


Configuration options shared across all sink types:

| Field | Description |
|--|--|
| `filter` | the minimum severity for log events to be emitted to this sink. This can be set to NONE to disable the sink. |
| `format` | the entry format to use. |
| `redact` | whether to strip sensitive information before log events are emitted to this sink. |
| `redactable` | whether to keep redaction markers in the sink's output. The presence of redaction markers makes it possible to strip sensitive data reliably. |
| `exit-on-error` | whether the logging system should terminate the process if an error is encountered while writing to this sink. |
| `auditable` | translated to tweaks to the other settings for this sink during validation. For example, it enables `exit-on-error` and changes the format of files from `crdb-v1` to `crdb-v1-count`. |



<a name="output-to-fluentd-compatible-log-collectors">

## Sink type: Output to Fluentd-compatible log collectors


This sink type causes logging data to be sent over the network, to
a log collector that can ingest log data in a
[Fluentd](https://www.fluentd.org)-compatible protocol.

{{site.data.alerts.callout_danger}}
TLS is not supported yet: the connection to the log collector is neither
authenticated nor encrypted. Given that logging events may contain sensitive
information, care should be taken to keep the log collector and the CockroachDB
node close together on a private network, or connect them using a secure VPN.
TLS support may be added at a later date.
{{site.data.alerts.end}}

At the time of this writing, a Fluent sink buffers at most one log
entry and retries sending the event at most one time if a network
error is encountered. This is just sufficient to tolerate a restart
of the Fluentd collector after a configuration change under light
logging activity. If the server is unavailable for too long, or if
more than one error is encountered, an error is reported to the
process's standard error output with a copy of the logging event and
the logging event is dropped.

The configuration key under the `sinks` key in the YAML
configuration is `fluent-servers`. Example configuration:

~~~ yaml
sinks:
   fluent-servers:        # fluent configurations start here
      health:             # defines one sink called "health"
         channels: HEALTH
         address: 127.0.0.1:5170
~~~

Every new server sink configured automatically inherits the configurations set in the `fluent-defaults` section.

For example:

~~~ yaml
fluent-defaults:
    redactable: false # default: disable redaction markers
sinks:
  fluent-servers:
    health:
       channels: HEALTH
       # This sink has redactable set to false,
       # as the setting is inherited from fluent-defaults
       # unless overridden here.
~~~

The default output format for Fluent sinks is
`json-fluent-compact`. The `fluent` variants of the JSON formats
include a `tag` field as required by the Fluentd protocol, which
the non-`fluent` JSON [format variants](log-formats.html) do not include.

{{site.data.alerts.callout_info}}
Run `cockroach debug check-log-config` to verify the effect of defaults inheritance.
{{site.data.alerts.end}}



Type-specific configuration options:

| Field | Description |
|--|--|
| `channels` | the list of logging channels that use this sink. See the [channel selection configuration](#channel-format) section for details.  |
| `net` | the protocol for the fluent server. Can be "tcp", "udp", "tcp4", etc. |
| `address` | the network address of the fluent server. The host/address and port parts are separated with a colon. IPv6 numeric addresses should be included within square brackets, e.g.: [::1]:1234. |


Configuration options shared across all sink types:

| Field | Description |
|--|--|
| `filter` | the minimum severity for log events to be emitted to this sink. This can be set to NONE to disable the sink. |
| `format` | the entry format to use. |
| `redact` | whether to strip sensitive information before log events are emitted to this sink. |
| `redactable` | whether to keep redaction markers in the sink's output. The presence of redaction markers makes it possible to strip sensitive data reliably. |
| `exit-on-error` | whether the logging system should terminate the process if an error is encountered while writing to this sink. |
| `auditable` | translated to tweaks to the other settings for this sink during validation. For example, it enables `exit-on-error` and changes the format of files from `crdb-v1` to `crdb-v1-count`. |



<a name="standard-error-stream">

## Sink type: Standard error stream


The standard error output stream of the running `cockroach`
process.

The configuration key under the `sinks` key in the YAML configuration
is `stderr`. Example configuration:

~~~ yaml
sinks:
   stderr:           # standard error sink configuration starts here
      channels: DEV
~~~

{{site.data.alerts.callout_info}}
The server start-up messages are still emitted at the start of the standard error
stream even when logging to `stderr` is enabled. This makes it generally difficult
to automate integration of `stderr` with log analyzers. Generally, we recommend using
[file logging](#output-to-files) or [network logging](#output-to-fluentd-compatible-log-collectors)
instead of `stderr` when integrating with automated monitoring software.
{{site.data.alerts.end}}

It is not possible to enable the `redactable` parameter on the `stderr` sink if
`capture-stray-errors` (i.e., capturing stray error information to files) is disabled.
This is because when `capture-stray-errors` is disabled, the process's standard error stream
can contain an arbitrary interleaving of [logging events](eventlog.html) and stray
errors. It is possible for stray error output to interfere with redaction markers
and remove the guarantees that information outside of redaction markers does not
contain sensitive information.

For a similar reason, no guarantee of parsability of the output format is available
when `capture-stray-errors` is disabled, since the standard error stream can then
contain an arbitrary interleaving of non-formatted error data.



Type-specific configuration options:

| Field | Description |
|--|--|
| `channels` | the list of logging channels that use this sink. See the [channel selection configuration](#channel-format) section for details.  |
| `no-color` | forces the omission of VT color codes in the output even when stderr is a terminal. |


Configuration options shared across all sink types:

| Field | Description |
|--|--|
| `filter` | the minimum severity for log events to be emitted to this sink. This can be set to NONE to disable the sink. |
| `format` | the entry format to use. |
| `redact` | whether to strip sensitive information before log events are emitted to this sink. |
| `redactable` | whether to keep redaction markers in the sink's output. The presence of redaction markers makes it possible to strip sensitive data reliably. |
| `exit-on-error` | whether the logging system should terminate the process if an error is encountered while writing to this sink. |
| `auditable` | translated to tweaks to the other settings for this sink during validation. For example, it enables `exit-on-error` and changes the format of files from `crdb-v1` to `crdb-v1-count`. |




<a name="channel-format">

## Channel selection configuration

Each sink can select multiple channels. The names of selected channels can
be specified as a YAML array or as a string.

Example configurations:

~~~ yaml
# Select just these two channels. Space is important.
channels: [OPS, HEALTH]

# The selection is case-insensitive.
channels: [ops, HeAlTh]

# Same configuration, as a YAML string. Avoid space around comma
# if using the YAML "inline" format.
channels: OPS,HEALTH

# Same configuration, as a quoted string.
channels: 'OPS, HEALTH'

# Same configuration, as a multi-line YAML array.
channels:
- OPS
- HEALTH
~~~

It is also possible to select all channels, using the "all" keyword.
For example:

~~~ yaml
channels: all
channels: 'all'
channels: [all]
channels: ['all']
~~~

It is also possible to select all channels except for a subset, using the
"all except" keyword prefix. This makes it possible to define sinks
that capture "everything else". For example:

~~~ yaml
channels: all except ops,health
channels: all except [ops,health]
channels: 'all except ops, health'
channels: 'all except [ops, health]'
~~~