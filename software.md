# Software

Once done with the hardware, there's a few things you should have a look at on the software level before you install Elasticsearch.

#### The Linux \(or FreeBSD\) kernel {#2262}

The Linux kernel version you run on is important. Prefer the 4.9.x version as it provides many interesting fixes and performances improvements, and comes with a great tracing framework you can rely on when troubleshooting performances. Take the time to read the Kernel CHANGELOG every time a new release is out, it and don't mind upgrading often if it's worth it. For example, Linux 4.9.25 \(or 4.9.0.0-BPO3 on Debian\) fixed many regressions on XFS introduced with the 4.9.13 that screwed up the filesystem performances.

#### The Java Virtual Machine {#4766}

The JVM you pick up is critical too, and you need to read and understand their specifications to get the best of your cluster. Elasticsearch runs best on Java 1.8, which provides G1GC, and does not support the unreleased Java 1.9 yet, but it supports various flavors of the Java virtual machine, so chose wisely. I usually run the Oracle JVM, but OpenJDK is cool too. Once again, don't mind upgrading your Java version often if a release fixes bugs of improve performances. Elasticsearch is not MySQL, and rebooting often is OK. Read the CHANGELOG to stay up to date.

#### The filesystem {#c616}

Last but not least, choosing the filesystem is a critical part of designing an Elasticsearch cluster. With small datasets on your disks, Ext4 is a fast, reliable option that does not need tuning. If you plan to store lots of indexes and more than 1TB of data per node, prefer a well tuned XFS for better performances.

