# The basics
What is iportant for your environment really depends on what and how much you are monitoring, how your pollers are distributed, the network speed, disk speed, CPU, RAM ...
But even if there is no clear answer to that, i can give a few hints that were helpful for tuning the performance in our environment.
It might contain information that could be essential for your situation.

To understand further explanations better, it might be a good idea to know a few things about how cacti works.

## What's cacti doing internally?
Let's begin with the rough basics about how the data is internally flowing, then you will understand this topic step by step.

When cacti processes data, it is doing that in several steps, beginning with the retrieval of the data from the devices.
If you use remote polling, which is very likely when you have a larger setup, this is happening mostly on the remote poller systems.
This is almost only affected by processing speed of the poller, plus a bit by the poller's DB performance, when the process first asks for a list of devices to be polled.
So you should use some halfway decent CPUs here that can keep up with all the little bits that are going on in the background.

The next step is the propagation of those values into the central database.
If you are operating multiple pollers, it really helps to get rid of the flood of data as fast as possible.
First reception is happening in a RAM table, which is halfway uncritical unless you configured your poller_output table as "on disk".

Then the data is postprocessed and written into the disk-backed poller_output_boost table.
This already takes quite a bit of Disk I/O, even if the transfer happens in chunks by the internal mechanisms in MySQL.
The chunks of data are usually larger, therefore we are fine with a high thruput of the storage devices at this point.

At some point in time, this output table is closed, no further data is added to it, and the final step of data processing begins.
The data is now transferred into lots of smaller files (RRDs - round robin databases) which contain only the data for a specific small set of data sources.
Those files are the final storage for your metrics.

Now when you request a graph to be plotted, cacti does a bit of DB-magic to gather all the bits and pieces that are needed to create your desired diageram and then it calls "rrdtool" to read the RRD files and plot the graph.

