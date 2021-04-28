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

The following chapters describe the components and their performance considerarions.

 * [The Basics ](Basics.md) is describing a few things inside cacti, to understand the tuning considerations.
 * [The Hardware](Hardware.md) gives you some ideas to plan your hardware setup.
 * [The Database](Database.md) shall provide an overview about the database tuning.

