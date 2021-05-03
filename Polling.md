# Polling Optimizations
This chapter will cover considerations you should take if you are working with several remoter pollers.

There are a few factors that come into play when multiple pollers are delivering data to a single central database machine.
All those factors might introduce new sources for possible congestions, some of which you don't have in a single system setup.

## The symptoms
One big problem can occur when the poller cycle really takes too much time, exceeding the 300 seconds and your poller process gets killed.
This will leave you with many gaps in the graphs that are most likely to be at the end of your poller cycle.

Another problem can be an instability in the spine, making it actually crash.
This will also produce gaps and is also causing the poll time to be reported wrongly, as the cacti system seems to think this process is still running and it keeps waiting for it to finish.

Spine crashes can happen in a loaded environment, and you will see them if you enable the cacti error-logs.
In the database you can probably also see an increased count of aborted connections and/or aborted clients.
These aborted connections are usually just a side-effect that can be used as an indicator that something is going on in the environment.

## Possible sources for your problems
If you see the poll time peaking, you should start to look for some of the not so obvious sources.

### Parallel polls
When you are working in enterprise environments, it is likely that you are not the only team that wants to retrieve data from the devices.
If now several applications are trying to retrieve data from the same device at the same time, the device will react slower and a local poller process is blocked for some time until it actually sees the data or a timeout.
As a result, the poll time will rise for those devices, and this adds up to the total poll time for a poller.

### Fast queries vs. slow queries
One factor can be the data processing speed differences between SNMP and scripted queries.
Since SNMP polling is mostly done directly in spine, this will happen at a relatively high speed, as spine is completely written in C, which is among one of the languages providing fastest execution times, as the resulting binary files contain commands directly at OS-level, sometimes even at CPU-level. So all actions are processed in nearly the fastest possible way.

Scripted queries - depending on their actual data source - can involve a lot more overhead, because
 - for each polled set of metrics, the script needs to be started up first - this takes a bit of time
 - then the script is usually interpreted which is slower than bein directly executed as in a compiled binary
 - sometmies scipts also involve calling other scripts or shell commands which then do the actual data retrieval, this takes again additional time
 - some scripts use SNMP methods in the end again to retrieve the actual data, so these kind of scripts add some overhead
   (for example because you need to re-format tabellaric data so that cacti can process it, or you need to add multiple OIDs to obtain the actual data point)

So in total, some queries can run much slower than others, specially when you need to use scripts to process data that is actually coming from an SNMP source again.

If you now have a lot of fast data sources on your devices, it is most likely that those are continuously processed by all the parallel threads created by spine.
Usually the SNMP data sources are processed rather fast and deliver multiple answers, when using snmpwalk or bulk get methods - depending of course also on the processing speed of the remote devices.
But this means a lot of data will be hitting your database at the same time from these fast data sources, while scripted data sources can result in a much slower data flow, for example if the script needs to perform a log-in before the actual query commands can be executed on a remote device.

### Data-Inrush peaks
If you run just a single poller or a low number of remote pollers, or if you have just a fewhundret devices to poll, you will probably not see any negative effect here.
But if many poller processes are started on multiple machines at exactly the same time, then you will most likely see a significant initial peak of connections and also row activities, which will decrease after a bit of time, until most of the faster data sources are processed and only the slower queries are delivering their data.

This can be fine-tuned a bit with the processes and threads parameters, but it can be a nuisance finding the correct tuning, because you need to take into account several constraints like DB connections, also polling time in general (less parallel threads = longer processing time).

### Database transactions and locked tables
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
If we have a lot of pollers sending data initially fast and considering that only one data row at a time can in the end be processed by the database, the logical consequence is that the processing is significantly slower as all these little things are adding up and also influence each other.
We had observed in our environment, that if one poller is peaking in polling-time, several others are also starting to peak.
And after really months of investigations, fine-tunings and also some cacti upgrades, this puzzle was finally solved after we gor more and more overview about which small things are going on in the background.

## The way out of it
First of all, database tuning is definitely one way to go, but usually that will only help to improve a few of the symptoms you see.

Also tuning the threads/processes settings can be a good idea but specially with many pollers, this can turn out to be a lengthier process with only slight improvements and some side-effects.

When it comes to parallel polling from several machines, the final solution can be to shift the poller time, actually starting the pollers at slightly different points in time.
Start with the poller which has the most devices and losts of relatively fast SNMP based metrics and just delay the poller start by 10-20 seconds just by adding a sleep to the cron job.
If this helps, it might be worth considering to write a wrapper script which you can configure easily per poller.
This can help relaxing the in-rush of data to the main database a bit, hence distributing the core load across a wider time span.
As a final solution it makes sense to start the smaller pollers later than the most loaded ones, because the smaller ones finish in less time, so you can delay them without having to worry too much about the poller cycle time of the whole setup.
