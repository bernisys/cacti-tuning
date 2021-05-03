# Polling Optimizations

This chapter will cover considerations you should take if you are working with several remoter pollers.

There are a few factors that come into play when multiple pollers are delivering data to a single central database machine.
All those factors might introduce new sources for possible congestions, which you don't have in a single system setup.

## Fast queries vs. slow queries
One factor is the data processing speed differences between SNMP and scripted queries.
Since SNMP polling is mostly done directly in spine, this will happen at a quite high speed, as spine is completely written in C, which is among one of the languages providing fastest execution times, as the resulting binary files contain commands directly at OS-level, sometimes even at CPU-level. So all actions are processed in nearly the fastest possible way.

Scripted queries - depending on their actual data source - can involve a lot more overhead, because
 - for each polled set of metrics, the script needs to be started up first - this takes a bit of time
 - then the script is usually interpreted which is slower than bein directly executed as in a compiled binary
 - sometmies scipts also involve calling other scripts or shell commands which then do the actual data retrieval, this takes again additional time
 - some scripts use SNMP methods in the end again to retrieve the actual data, so these kind of scripts add some overhead
   (for example because you need to re-format tabellaric data so that cacti can process it, or you need to add multiple OIDs to obtain the actual data point)

So in total, some scripts can run much slower than the SNMP queries, specially when you use scripts to process data that is actually coming from an SNMP source again.

If you now have a lot of fast data sources on your devices, it is most likely that those are continuously processed by all the parallel threads created by spine.
Usually the SNMP data sources are processed rather fast and deliver multiple answers, when using snmpwalk or bulk get methods - depending of course also on the processing speed of the remote devices.
But this means a lot of data will be hitting your database at the same time from these fast data sources, while scripted data sources can result in a much slower data flow, for example if the script needs to perform a log-in before the actual query commands can be executed on a remote device.

## Data-Inrush peaks
If you run just a single poller or a low number of remote pollers, or if you have just a fewhundret devices to poll, you will probably not see any negative effect here.
But if many poller processes are started on multiple machines at exactly the same time, then you will most likely see a significant initial peak of connections and also row activities, which will decrease after a bit of time, until only the slower queries are delivering their data.

## Database transactions and locked tables
This is an effect that probably comes into play when multiple pollers are writing their data to the same table on the main poller.
This is from the aatabase point of view not a parallel activity and the data needs to be serialized first to make sure that there are no row activities that are mutually excluding.
As far as i have seen in the SQL queries and at some points in the cacti code, several write activities are done in transactions which collect many row updates into a single one.
Those transactions need to be prepared before they are executed and this takes a bit of time.
And when the data is actually written to the table, the table will be locked so that no other process can write data to it.
This is to avoid accidentally messing up the data.

Some transactions can take significant longer times than others, and the bigger your tables get, the more time is needed for specific operations in these tables.
Mainly two tables are significantly larger than all others, which are the boost table and the SNMP cache table.
Those can easily contain millions of entries.

## Conclusions from the above points
If we have a lot of pollers sending data initially fast and considering that only one data row at a time can in the end be processed by the database, the logical consequence 

