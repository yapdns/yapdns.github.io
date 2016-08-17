---
layout: post
title: Introducing YAPDNS
---

This summer I have been working on YAPDNS - Yet Another Passive DNS System.

## Passive DNS

> "Passive DNS" or "passive DNS replication" is a technique invented by Florian Weimer in 2004 to opportunistically reconstruct a partial view of the data available in the global Domain Name System into a central database where it can be indexed and queried.

\- [Farsight Security](https://www.farsightsecurity.com/Technical/Passive_DNS/)

## Existing Solutions

There are a couple of tools out there to collect Passive DNS data (e.g. passivedns by gamelinux and pdnsd), but they only work by sniffing authoritative DNS answers inside network traffic and by storing them. 

There is a huge amount of other sources that could be used to collect Passive DNS data: for example, almost every organization has a web proxy or gateway, and its logs almost always contain a domain name, an IP address and a timestamp. The same data set can be extracted from other textual logs from DNS servers (Bind, Microsoft DNS, etc), web servers, IDS/IPS, and even sandboxes (Cuckoo) and honeypots (Thug) or other Passive DNS databases (VirusTotal, DNSDB, etc). YAPDNS provides an interface to collect basic associations between an IP address and a domain name, along with the first and last time the association was seen. Other data can be added for specific log sources (e.g. DNS logs also contain TTL, record type, etc), or gathered from external repositories (e.g. association with malware in VirusTotal’s database, etc).

## How does YAPDNS work?

### YAPDNS Client [[Github](https://github.com/yapdns/yapdnsbeat)]

The YAPDNS Client (or YAPDNSBeat) is responsible for watching log files, extracting DNS records and sending them to YAPDNS application. The client is a fork of filebeat a project started by elastic - to index logs into elasticsearch, logstash and other data stores. The project really fit well into our requirements - Filebeat already had a robust system design to watch files concurrently using goroutines and managed the configuration for us. Filebeat is based on libbeats library which allowed us to extend it by adding our own publisher module that would write data to backend server HTTP API Endpoint.

Since we client could submit the same DNS entry repeatedly we wanted to do some caching - to avoid indexing the same DNS records again. So we added a caching layer in the Spooler - just before writing the DNS record to the backend.

### YAPDNS Application [[Github](https://github.com/yapdns/yapdns-app)]

The YAPDNS application is a Django application responsible for serving API endpoint for YAPDNS clients and for the dashboard used to correlate and analyze the DNS data. We use Postgres for storing the client related data and elasticsearch for storing DNS records. Elasticsearch being a distributed data store - gives us the flexibilty to scale easily in future and also allows to perform filters and aggregations on our data.

The application uses Docker and Docker Compose for deployment. With 4 containers - postgres, elasticsearch, python/django application and nginx.

### DNSDump Reporting Module for Cuckoo [[Github](https://github.com/yapdns/cuckoo-dnsdump)]

To test YAPDNS client - we needed to deploy it on a host running proxy / mail / dns server that would generate logs with meaningful DNS data. We thought we could leverage existing cuckoo servers that already collect lot of networking data. So we built a reporting module that would dump dns records in tab separated file for each of the analysis.

The module takes in `output_dir` as parameter - which is the directory where all the dns dumps are placed. To test the client we ran the client and set it watch the folder with all the DNS logs.

## Future plan

The Django application right now is still barebones - it does enough just to index the data into elasticsearch and a minimal frontend that lists DNS records for a particular domain logged by the clients. There is lot of numbers that can be crunched from this data and coming months we'll be working on developing a dashboard that serves interesting visualizations from the data collected.

Other things - 
 
1. HTTPS support
2. Background workers to add GeoIP information to DNS records
3. Authentication for the dashboard
