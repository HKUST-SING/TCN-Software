#1 What is TCN
TCN is a new ECN marking scheme. It can work with any packet scheduling algorithm. Currently, we have implemented TCN on the top of two schedulers: SP/DWRR and SP/WFQ. In both schedulers, we have several priorities. Queues on the same priority are scheduled based on DWRR (Deficit Weighted Round Robin) or WFQ (Weighted Fair Queueing). Queues from different priorities are scheduled based on strict priorities (SP).  

#2 How to use
##2.1 Compiling
TCN software prototype is implemented as a Linux queuing discipline (qdisc) kernel module, running on multi-NIC servers to emulate switch hardware behaviors. So you need the kernel headers to compile it. You can find available headers on your system in `/lib/modules`. To install the kernel headers, you just need to use the following command：
<pre><code>$ apt-get install linux-headers-$(uname -r)
</code></pre>

To compile SP/DWRR kernel module:
<pre><code>$ cd sch_dwrr
$ make
</code></pre>
This will produce a kernel module called `sch_dwrr.ko`. 

To compile SP/WFQ kernel module:
<pre><code>$ cd sch_wfq
$ make
</code></pre>
This will produce a kernel module called `sch_wfq.ko`.

##2.2 Installing
TCN replaces token bucket rate limiter module. Hence, you need to remove `sch_tbf` before installing TCN. To install TCN (e.g., SP/DWRR) on a device (e.g., eth1):

<pre><code>$ rmmod sch_tbf
$ insmod sch_dwrr.ko
$ tc qdisc add dev eth1 root tbf rate 995mbit limit 1000k burst 1000k mtu 66000 peakrate 1000mbit
</code></pre>

To remove TCN (SP/DWRR) on a device (e.g., eth1):
<pre><code>$ tc qdisc del dev eth1 root
$ rmmod sch_dwrr
</code></pre>

In above example, we install TCN (SP/DWRR) on eth1. The shaping rate is 995Mbps (line rate is 1000Mbps). To accurately reflect switch buffer occupancy, we usually trade a little bandwidth. 


##2.3 Note
To better emulate real switch hardware behaviors, we should avoid large segments on server-emulated software switches. Hence, we need to disable related offloading techniques on all involved NICs. For example, to disable offloading on eth0: 
<pre><code>$ ethtool -K eth0 tso off
$ ethtool -K eth0 gso off
$ ethtool -K eth0 gro off
</code></pre>

##2.4 Configuring
Except for shaping rate, all the parameters of TCN are configured through `sysctl` interfaces. Here, I only show several important parameters. For the rest, see `params.h` and `params.c` for more details.

<ul>
<li>
ECN marking scheme for SP/DWRR:
<pre><code>$ sysctl dwrr.ecn_scheme
$ sysctl wfq.ecn_scheme
</code></pre>
To enable per-queue ECN marking:
<pre><code>$ sysctl -w dwrr.ecn_scheme=1
$ sysctl -w wfq.ecn_scheme=1
</code></pre>
To enable per-port ECN marking:
<pre><code>$ sysctl -w dwrr.ecn_scheme=2
$ sysctl -w wfq.ecn_scheme=2
</code></pre>
To enable <a href="http://sing.cse.ust.hk/~wei/papers/mqecn-nsdi2016.pdf">MQ-ECN</a> (this has no use for SP/WFQ):
<pre><code>$ sysctl -w dwrr.ecn_scheme=3
</code></pre>
To enable TCN:
<pre><code>$ sysctl -w dwrr.ecn_scheme=4
$ sysctl -w wfq.ecn_scheme=4
</code></pre>
To enable CoDel ECN marking:
<pre><code>$ sysctl -w dwrr.ecn_scheme=5
$ sysctl -w wfq.ecn_scheme=5
</code></pre>
</li>

<li>Per-port ECN marking threshold (bytes):
<pre><code>$ sysctl dwrr.port_thresh
$ sysctl wfq.port_thresh
</code></pre>
</li>

<li>Per-queue ECN marking threshold (bytes) (i is index starting from 0):
<pre><code>$ sysctl dwrr.queue_thresh_i
$ sysctl wfq.queue_thresh_i
</code></pre>
</li>

<li>Per-port shared buffer size (bytes):
<pre><code>$ sysctl dwrr.shared_buffer
$ sysctl wfq.shared_buffer
</code></pre>
</li>

<li>TCN sojourn time threshold (1024 ns):
<pre><code>$ sysctl dwrr.tcn_thresh
$ sysctl wfq.tcn_thresh
</code></pre>
</li>
</ul>
