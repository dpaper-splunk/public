# Index Usage Reporting 

## What is it?

The IUR (“Index Usage Reporting”) app is intended to assist Splunk administrators in deciding which data to continue ingesting into Splunk by way of examining if an index is ever searched, and if so, how often.

IUR provides information about index search usage that is not available in Monitoring Console.

## Where is IUR getting this data from? 

The data IUR uses first appeared in Splunk Enterprise 9.0 in the Search Head’s internal logging for a search. Search.log was enhanced in 9.0 to include information about all indexes that are in scope for a specific search.
 
## Installation info

- Installed on the Search Head
- Contains configs to capture data relevant to indexes being searched
- Search Head should be forwarding data for its internal logging, including now search.log, to the indexing tier
- A new sourcetype is created and data is sent to _internal, so it does not affect metered ingest licensing

## Versions Supported

- 9.0

## Deployment Supported

- Splunk Enterprise (on-premise)

IUR running in Splunk Cloud is not supported as of October 2023.

## How does it work?

### Ingestion

IUR depends on the `search.log` for each search run on a Search Head. The `search.log` data is found inside each `$SPLUNK_HOME/var/run/splunk/dispatch/` directory and is present for every search executed on the search head. Every search gets its own directory entry in dispatch. These directories are reaped out when the Time To Live for that search result expires. IUR includes an `inputs.conf` that ingests each searches `search.log` as it’s created.

### The Data
IUR works by using the data in `search.log` that now exists to identify expanded lists of indexes being searched when a user includes a wildcard in the `index=` in their SPL. Examples include `index=*`, `index=prod*`, `index=*_logs` or `index=_*`. 

In the example below, this is what that expansion looks like for a search using wildcards in `index=`

    10-06-2023 10:14:16.931 INFO  SearchParser [18820582 RunDispatch] - PARSING: search index=_* | stats count
    ...
    10-06-2023 10:14:17.340 INFO IndexScopedSearch [18820613 localCollectorThread] - IndexScopedSearch is called for index = _internal, et = 1694008756.000000000, lt = 1694009656.000000000, index_et =
    10-06-2023 10:14:17.325 INFO IndexScopedSearch [18820613 localCollectorThread] - IndexScopedSearch is called for index = _audit, et = 1694008756.000000000, lt = 1694009656.000000000, index_et =
    10-06-2023 10:14:17.346 INFO IndexScopedSearch [18820613 localCollectorThread] - IndexScopedSearch is called for index = _introspection, et = 1694008756.000000000, lt = 1694009656.000000000, index_et =
    10-06-2023 10:14:17.333 INFO IndexScopedSearch [18820613 localCollectorThread] - IndexScopedSearch is called for index = _configtracker, et = 1694008756.000000000, lt = 1694009656.000000000, index_et =
    10-06-2023 10:14:17.353 INFO IndexScopedSearch [18820613 localCollectorThread] - IndexScopedSearch is called for index = _telemetry, et = 1694008756.000000000, lt = 1694009656.000000000, index_et =
    10-06-2023 10:14:17.363 INFO IndexScopedSearch [18820613 localCollectorThread] - IndexScopedSearch is called for index = _thefishbucket, et = 1694008756.000000000, lt = 1694009656.000000000, index_et =

IUR works for searches that don't include an `index=` at all.

    10-06-2023 10:34:58.548 INFO  SearchParser [10623692 searchOrchestrator] - PARSING: search sourcetype=metrics | stats count
    …
    10-06-2023 10:34:58.825 INFO  IndexScopedSearch [10623692 searchOrchestrator] -  IndexScopedSearch is called for index = _thefishbucket, et = 1697378400.000000000, lt = 1697466898.000000000, index_et = -9223372036854775808.000000000, index_lt = 9223372036854775807.999999000, noRead = FALSE
    10-06-2023 10:34:58.834 INFO  IndexScopedSearch [10623692 searchOrchestrator] -  IndexScopedSearch is called for index = avsname, et = 1697378400.000000000, lt = 1697466898.000000000, index_et = -9223372036854775808.000000000, index_lt = 9223372036854775807.999999000, noRead = FALSE
    10-06-2023 10:34:58.846 INFO  IndexScopedSearch [10623692 searchOrchestrator] -  IndexScopedSearch is called for index = main, et = 1697378400.000000000, lt = 1697466898.000000000, index_et = -9223372036854775808.000000000, index_lt = 9223372036854775807.999999000, noRead = FALSE

By looking for the value after `index = ` in the search.log entries above, we can generate the analytics Splunk administrators have been asking for.

## Notes

Unless the Splunk environment is already ingesting `search.log` files from dispatch on the Search Head, this app will take time to produce reliable data. On install and restart, all existing dispatch directories are read, and the initial set of indexes being searched within the TTLs of recently completed searches show up. However, the data really needs to be collected for a longer period of time to get a proper view of what is being searched and what isn’t. 

Suggested best practice for this is to collect a minimum of 3 months of `search.log` data, preferrably 6 months, to account for potential long windows of time between uses of data for certain indexes. Examples of why so long include 

- Teams that use Splunk only when there are production issues which will happen with indeterministic frequencies depending on the environment
- Teams that use the data only when there is a new code install to validate things are working and installs happen only every 6 weeks or once a quarter
- Data for compliance and audit reporting that is used infrequently, but needs to be there
