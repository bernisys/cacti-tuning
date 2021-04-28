# cacti-tuning
A bit about cacti DB and other tuning possibilities that might help in larger setups.

## Introduction 
First of all, if i am talking about a larger setup, i am talking about several remote poller systems, thousands of hosts and hundretthousands of data sources.
This is the setup used in a datacenter of a company i am working for (in 2021-04):
 * one failover cluster with two servers as master poller, 2x10 core CPUs + HT, 64GB RAM, SSD-backed storage
 * several remote systems, usually VMs with 4 vCores and 16GB RAM
 * about 3500 network devices of different vendors, classes and types (switches, routers, firewalls, load balancers, several other devices)
 * 450.000 graphs implemented
 * 900.000 data sources

## The basics
What is iportant for your environment really depends on what and how much you are monitoring, how your pollers are distributed, the network speed, disk speed, CPU, RAM ...
But even if there is no clear answer to that, i can give a few hints that were helpful for tuning the performance in our environment.
It might contain information that could be essential for your situation.

To understand further explanations better, it might be a good idea to know a few things about how cacti works.

### What's cacti doing internally?
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

Now when you request a graph to be plotted, cacti calls "rrdtool" to read the RRD files and plot the graph.


## The hardware 
Coming now to the first important thing that you should consider:

What kind of hardware should i use?

### Main server
This machine does a lot of data processing in the central DB and can produce a rather high CPU and disk load.

Some pointers that might help with your decision:
  * Use a powerfull machine with enough RAM and CPU power fitting our needs.
  * It helps using physical hardware, as it doesn't clash with other load
  * Multi-Core machines are a must, simply to satisfy the central DB
  * Give it enough RAM - comparison, the above mentioned setup was using a DB buffer size of more than 20GB
  * If you think you have enough RAM on the main DB - buy more, you won't regret it
  * Use SSD backed storages, either locally or in a SAN environment
  * If M.2 is possible, go for that one, it's faster than SAS in terms of IOPS, and that is what you will need

#### Why can't we use a VM?
Well, I didn't say you can't ... i only recommend to not do that. :)

There are probably setups where a VM is keeping up fine with the load, specially for small to mid size environments with a few hundret devices and 100k-200k data sources.
While a VM can be pretty helpful when it comes to software upgrades (snapshots), it has a few drawbacks, which all have to do with the VM usually not being the only one in the cluster. And when ever the cluster load is high, it is likely affecting your VM in many ways.

Here's a short overview about things that are not beneficial in a virtualized environment:
 * a VM usually shares CPUs with other VMs, and the more cores you assign the more impact you might feel
 * IF you really want to try with a VM, maybe for financial reasons (hey the cluster is already there), go with core-binding!
 * a VM shares its RAM with other VMs, and if there is a RAM congestion in the cluster, then you are likely to experienve surge of slowness in yout VM
 * a VM shares the storage with other VMs.
   This is not a problem with the storage or the cluster, but more with the underlying network components.
   Usually you would use a SAN setup, which involves multiple multi-gigabit links which might work fine most of the time.
   But specially in times where other VMs overload that network, you are likely to see bit of a performance degradation.

#### Why so much RAM?
Well, you have to deal with a whole lot of data.
Our setup is producing about 16 million data rows in the boost table within less than two hours.
All this data needs to be processed as quickly as possible, so that it is not accumulating and filling up your DB over time.
If you have a high amount of RAM, the DB can react much quicker compared to having to fetch and stora all data from and to a harddisk.
Even if you use SSDs, they will be at their limits at some point, and this is when your environment will completely break down, as the main DB is the most essential part of cacti.

#### What's with the IOPS?
IOPS stands for Input/Output operations per second, and it is a highly important performance value for harddisks.
This is not to be confused with the actual data thruput of your disks!
IOPS describe more or less the capability of your harddrive to accept and execute tasks that the operating system requests.
The higher the number, the more tasks it can process per second.
 * Classical HDDs are usually limited at 250..500 or maybe even up tp 1000 in high-spinning enterprise-class harddisks.
   Even if they can deliver a quite healthy consistent data stream, they totally suck when it comes to addressing a completely different part of the disk.
   This involves positioning a magnetic arm and waiting for the correct portion of data rushing thru under the read/write head.
 * SATA based SSDs are way faster, ususally starting around 15.000 and going up to 75k .. 150k.
   Those disks have no moving parts, so there is no need for mechanical positioning - this makes them blazingly fast already when addressing the data.
 * M.2 SSD cards exceed this value even more, sometimes up to a million (yes, 1.000.000).
   This is mainly because they use a different interface which is much faster than the serial line used by SSDs.

### The pollers
Usually these machines can be much smaller, specially if you go for a higher amount of pollers which all just process a small portion of your devices.
You can for example provision them across your different data centers or place a poller into a specific part of your network, that's completely up to your needs and imagination.
You should try to divide the load halfway equally across the pollers, so that each of them sees about the same amount of devices or data sources.

Some notes that might help with your decisions:
 * You can use VMs here, as they only see a small share of the load that the main server will see.
 * CPU/Core count is Depending on the device count that is monitored with that poller.
   If you're flexible enough and can change this setting at any time, start with 2-4 cores.
 * The amount of RAM depends also strongly on the number of devices.
   If you are flexible, start with 8GB, see how it fits your need
 * Storage performance is not that important on a remote poller, as long as the whole poller DB fits into the RAM.

Hint: You can add CPU cores and RAM on the fly if your VM is configured for memory hot-add!

### Databases in NUNMA setups
When you are using a multi-CPU based system, the RAM is not shared between those CPUs as you might think.
Each physical bar of RAM is merely assigned to one physical CPU.
If the other CPU requests some content on a RAM section which the other CPU owns, it needs to be fetched by that CPU and copied over to the other one.
So both CPUs are blocked at the time of access and this can significantly affect your system's performance.

There are two nice articles by Jeremy Cole which i have found on the net which talk about exactly this problem:

 * https://blog.jcole.us/2010/09/28/mysql-swap-insanity-and-the-numa-architecture/
 * https://blog.jcole.us/2012/04/16/a-brief-update-on-numa-and-mysql/


