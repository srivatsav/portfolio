---
title: "The 99.9999999% Durability"
date: 2023-10-08T00:00:00+05:30
draft: false
tags: ["distributed-systems", "erasure-coding", "s3", "storage"]
---

*Originally published on [Medium](https://medium.com/@vatsav.gs/the-99-9999999-durability-d12373b29078)*

The motivation for this blog post was my random thoughts about the application of linear algebra in the computing world, especially in distributed systems. It started when I wondered what it would take to build a storage cloud service similar to S3.

## Why, What of Data Durability and Availability

Data is precious and it is irreplaceable and it should be available always. This drives the need for fault-tolerant high durability and high availability storage.

A common metric to represent durability and availability is using the number of nines.

For a durability of five nines (99.999%) it represents 0.99999 * 100,000 = 99,999 i.e 1 file for every 100,000 files is lost.

For a durability of nine nines (99.9999999%) it represents 0.999999999 * 100,00,00,000 = 99,99,99,999 i.e 1 file for every 100 million files.

For availability of five nines (99.999%) it represents 0.99999 * 1440 * 365 = 5,25,594 minutes i.e almost 6 minutes of downtime per year.

For a durability of nine nines (99.9999999%) it represents 0.999999999 * 1440 * 365 = 5,25,599.99 i.e almost 600 milliseconds of downtime.

To achieve a high number of nines storage systems must have high MTTF (Mean Time To Failures) and when a failure occurs the systems must have fast recovery times or low mean time to recover. The redundant way of achieving this fault tolerance is to do …

## Data Mirroring (majority of RAID levels)

Divide the data into fragments and write the fragments into separate failure domains. Then replicate the identical copies of those fragments across let's say 3 different nodes. So for data object of 4 fragments that would be mirrored across 3 nodes and 4 disks each. You definitely need not worry about this as this works for 100% and this strategy is absolutely foolproof. The only concern here is we're just setting Elon Musk's flamethrower to 'data mode' and watching those bills 💵 go up in smoke :-P

**Problem**: I am spending more money  
**Solution**: Linear Algebra ;-P

## What is Erasure Coding

Erasure coding is a mathematical algorithm that protects data by creating redundant pieces of information, known as "parity fragments", from the original data fragments. Instead of replicating the entire data, erasure coding splits the data into fragments, then creates additional parity fragments using matrix operations.

The key insight is that you can reconstruct the original data from any subset of fragments, as long as you have enough of them. This provides the same durability guarantees as full replication but with significantly less storage overhead.

For example, with a 4+2 erasure coding scheme:
- Original data is split into 4 data fragments
- 2 parity fragments are computed using linear algebra
- You can lose any 2 fragments and still reconstruct the data
- Storage overhead is only 50% instead of 200% with 3x replication

This is how AWS S3 achieves 99.999999999% (11 nines) durability while keeping costs manageable.