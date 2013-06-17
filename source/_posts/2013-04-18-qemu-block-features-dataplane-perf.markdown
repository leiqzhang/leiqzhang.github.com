---
comments: true
date: 2013-04-18 13:00:31
layout: post
slug: dataplane-perf
title: 'QEMU Block Fetures: dataplane perf'
categories:
- QEMU
- Virtualization
tags: [qemu, kvm, libvirt, dataplane, perf]

---

### Dataplane vs Non-dataplane Perf

1. Envrionment
    1. Qemu 1.4 master branch
    2. kernel: 3.5.0-2.fc17.x86-64
    3. virtual disks location: the same local SATA harddisk with ext4 fs
    4. VM start cmd (os: win7/qed, disk1: raw/non-dataplane/10G/NTFS, disk2: raw/dataplane/10G/NTFS) :
2. Testing Tool And Testing Params:
    1. Only IOMeter App Running in VM, and no other IO in Host (Dom0)
    2. Main Testing VM: 8 worker, 16K IO size, 0% Read,  100% Random,  and 50 outstanding IO 
    3. Pression VM: 8 worker, 16K IO size, 25% Read, 100% Random,  and 50 outstanding IO
3. Testing Results:

    |                | IOPS | MBPS  |
    |   :------:     | :------: | :-----:   |
    |dataplane       |  230.956328  |   3.608693  |
    |non-dataplane   |  178.324867  |   2.786326  |
