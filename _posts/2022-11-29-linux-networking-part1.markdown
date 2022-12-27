---
layout: post
title:  "Linux Kernel Networking - part 1: The Network Interface Card"
date:   2023-01-01 19:27:21 +0200
categories: Linux Networking
toc: true
---

## Introduction

This post is the first in a series of going through the Linux TCP/IP stack. The
point-of-view will be to cover what's useful for an application programmer wanting
to squeeze out the last bits of performance of their application. I will use a
`5.10.147-133.644` of the Amazon Linux 2 kernel and assume an EC2 instance with an
Elastic Network Adapter. This happens to be the setup that I use at work.

The first part will cover the NIC. We will go over the source code of the AWS Elastic
Network Adapter and how it interfaces with the kernel. From there we will work our way
up the TCP/IP stack. The next part of the series will look at the next layer, which is
XDP and setting up kernel bypass. In other words, what happens if we choose not to use
the TCP/IP stack.

The source code analyzed for the blog post can be found here:

* [Amazon Linux 2 kernel][linux-kernel]
* [Amazon ENA driver][ENA-driver]

The above links point to the precise source code running on the EC2 instances that can
be installed with the terraform code provided below.

## Setting Up a Test Environment

A great way to understand what happens inside the kernel is to analyze sampled kernel
stack frames as a [flamegraph][flamegraph]. These will instantly reveal the main code
paths taken  and helps visualize how packets travel through the kernel.

In order to capture stack frames, we need a test environment with the kernel and driver
version being analyzed. Terraforms for setting up the test environment and the load tester
I'm using can be found here:

* [Terraforms for test environment][test-env]
* [iperf3 network tester][iperf3]

The terraform code creates a complete VPC with only two EC2 instances in a cluster
placement group. There are two EC2 instances, a smaller for running the kernel tracing
tools, and a larger one for running the stress tester. Note that running this stuff is
not free.

The provided terraform script outputs the EC2 machine IDs. For example:

```
Apply complete! Resources: 16 added, 0 changed, 0 destroyed.

Outputs:

source = "i-0103eb2cb9de3f923"
source_ip = "3.208.27.174"
target = "i-09a3489eded5cd52b"
target_ip = "52.207.225.111"
```

I have the following snippet in my `~/.ssh/config`

```
Host i-*.* mi-*.*
  ProxyCommand bash -c "aws ssm start-session --profile $(echo %h|/usr/bin/cut -d'.' -f2) --target $(echo %h|cut -d'.' -f1) --region us-east-1 --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"
```

which allows me to log into a host with a command like the following:

```
ssh efagerho@i-0103eb2cb9de3f923.testing
```

Here `testing` is the named AWS profile that I use as the account and which is
configured in `~/.aws/(config|credentials)`.

The same also works with `scp`, which makes it easy to copy stacktrace dumps
from the hosts in order to generate flamegraphs etc.

## ENA Hardware Features

Elastic Network Adapter is the NIC used on Nitro EC2 instances. ENA hardware supports
a few key features:

1. [Checksum offload][checksum-offload]
2. [TCP transit segmentation offload][TSO] (TSO)
3. [Receive-side scaling][RSS] (RSS)

Note that the ENA provides at least 10Gb/s of bandwidth and depending on the instance
type can provide up to 100Gbit/s. Without the above features, achieving such
performance would not be possible.

The list of all enabled offloads supported by the device can be listed with

```
ethtool -k <DEVICE>
```

### Checksum Offloading

The [checksums][IP-checksum] used in the TCP/IP protocols are ones-complement sums of
all the 16-bits words in the header or data. If you would calculate a checksum of, say,
a buffer of 1024 bytes of data with the same algorithm, and you removed
the first 128 bytes, then you could deduce the checksum of the remaining bytes from
the original checksum and the checksum of the first 128 bytes. In other words, you do
not need to read again the remaining 896 bytes to calculate their checksum. Similarly,
if you modify any data in the buffer, you can deduce the change in the checksum from
your local modification alone. The advantage of this is that when computing checksums,
we only need to once compute the checksum of the whole data. We can then compute both
the IP and TCP header checksums by only inspecting the bytes in the IP and TCP headers.

As can be seen from the on-CPU flamegraph above memory bandwidth is a serious performance
bottleneck in packet processing. Therefore, we want to avoid wasting any precious
memory bandwidth. The question is then when to compute the checksum of the whole packet?
Note that the NIC has to inspect all the bytes in the packet, since it's responsible
for copying the data in the packet to RAM. Therefore, it makes sense for the NIC to
compute any checksums at this step and pass them on to the kernel. The kernel can
then avoid reading payload bytes during packet processing and only spend memory bandwidth
reading header bytes.

### Segmentation Offloading

Applications typically want to send large buffers to the kernel in one go, since it
reduces the number of syscalls that need to be made in order to send data.
On the other hand, network layers have Maximum Transmission Units (MTUs). Therefore,
larger chunks of data need to be split up into multiple packets.

TSO allows the kernel to send large buffers to the NIC. The NIC will then take care of
splitting the buffers into multiple smaller packets as well as create the appropriate
IP and TCP headers.

### Receive-side scaling

With modern multi-core CPUs the key to high-performance is to reduce contention by
ensuring cores don't write to the same data. Similarly, we need data locality to take
advantage of CPU caches.

RSS is another word for supporting multiple Rx/Tx queues. For TCP/UDP connections,
the NIC computes a hash of the ip/port pairs of the source and destination and
places the packet in one of the queues based on that.

With RSS we typically have one Rx/Tx queue per CPU. However, this is not entirely
accurate for EC2 instances with many cores as the maximum number of queues is
typically 8 for an ENI. You can check the maximum number of queues on the EC2 instance
with:

```
ethtool -l <DEVICE>
```

TODO: check for example instances.

TODO: mention that you can also reduce the number of queues with `ethtool -L`

This means that there's less contention for the same data and locks. Specially
engineered applications can take advantage of this by making sure that a thread
executing on a particular core only manages connections mapped to the Rx/Tx queues
on the same core. This allows the application to make optimum use of CPU caches
during packet processing. This can be achieved e.g. with the somewhat esoteric
SO_ATTACH_REUSEPORT_CBPF socket option. See for instance the following
[example][bpf-reuseport].

## Quick Stress Testing

Before digging into any code, let's stress test the TCP/IP stack and dig a little
into what happens. I'm using the following commands on the source
EC2 instance for `iperf3` to send traffic from the source to the target:

```
source: iperf3 -c 10.0.4.95
target: iperf3 -s
```

The IP is whatever my target EC2 instance happened to get. This will bombard the
target, which is a much smaller instance and fully max it out. In this particular
case, the test instanced have 10Gb NICs and `iperf3` can generate about 9.5Gb/s
of traffic, nearly saturating the interface.

For on-CPU profiles I use the following command on the target EC2 instance

```
sudo /usr/share/bcc/tools/profile -af -p 10441 10 > ~/oncpu.stacks
flamegraph.pl --hash --title="On-CPU Time Flame Graph" -width 1800 < oncpu.stacks > oncpu.svg
```

and for off-CPU profiles we use the command

```
sudo /usr/share/bcc/tools/offcputime -p 10441 -f 5 > offcpu.stack
flamegraph.pl --hash --title="Off-CPU Time Flame Graph" -width 1800 < offcpu.stacks > offcpu.svg
```

where `10441` is the PID of the iperf3 process on the target.

The on-CPU flamegraph is here

![image]({{site.url}}/assets/images/oncpu.svg)

and finally the off-CPU flamegraph here

![image]({{site.url}}/assets/images/offcpu.svg)

Looking at the on-CPU flamegraph, we see that the majority of the time is spent in
`tcp_recvmsg` and more than half the time in that function is spent in
`skb_copy_datagram_iter`, which copies the data from the kernel into userland.

Only a fraction of the time is spent in `ena_io_poll`, which is the part of the
driver that checks whether there is new data available.

## ENA Driver Internals

### NAPI Overview

NAPI is the "New API" for network device driver packet processing. Previously, NICs
would fire an interrupt on an incoming packet causing an interrupt handler in the
driver to run and push the packet to the TCP/IP stack. On a high-performance network
device, we can handle millions of packets per second, which would cause a storm of
interrupts. A CPU could simple never keep up with the load, since handling an
interrupt is a somewhat expensive operation.

With NAPI the kernel essentially polls a device. However, polling is wasteful if
there are no incoming packets, so this is done in a somewhat smarter way:

1. Initially, the device is configured to fire an interrupt on packet arrival.
2. When the first packet arrives, the interrupt handler turns on polling.
3. If the kernel polls the NIC and there are no packets available it turns interrupts back on.

The most important functions in the NAPI framework are:

1. [napi_schedule()][napi-schedule]: turns on device polling.
2. [napi_complete()][napi-complete]: turns off device polling.
3. [napi_poll()][napi-poll]: calls the driver I/O handler.

[napi_poll()][napi-poll] calls the driver I/O handler [ena_io_poll()][ena-io-poll].
This is where a packet is pushed from the driver to the next layer in the
network stack. What happens after that for inbound packets is the topic of the
follow-up blog post, so what remains on the Rx side is understand how and when
[ena_io_poll()][ena-io-poll] is called and what happens inside that function.

### Device Initialization

The module itself is initialized by a call to [ena_init()][ena-init] while the actual
device initialization work is done when [ena_probe()][ena-probe] is being called when
the PCI bus is scanned by the kernel. The latter also calls [register_netdev()][register-netdev]
which makes the device visible to the kernel (i.e. it shows up in `ifconfig`).
Finally, when the device is activate (e.g. running `ifconfig <DEVICE> up`) the
kernel calls [ena_open()][ena-open]. [ena_open()][ena-open] registers interrupt
handlers for the device and registers its Rx/Tx queues with the kernel.

Every driver allocates a data structure that is used to store all the device
information. For the ENA driver, this data structure is called
[ena_adapter][ena-adapter-struct]. It contains lots of data, but a few points are worth
mentioning

```c
struct ena_adapter {
	...
	/* TX */
	struct ena_ring tx_ring[ENA_MAX_NUM_IO_QUEUES]
		____cacheline_aligned_in_smp;

	/* RX */
	struct ena_ring rx_ring[ENA_MAX_NUM_IO_QUEUES]
		____cacheline_aligned_in_smp;

	struct ena_napi ena_napi[ENA_MAX_NUM_IO_QUEUES];

	struct ena_irq irq_tbl[ENA_MAX_MSIX_VEC(ENA_MAX_NUM_IO_QUEUES)];
	...
}
```

The driver supports `ENA_MAX_NUM_IO_QUEUES` number of I/O queues. Each I/O queue
has its own interrupt and separate NAPI polling state. This is because the
NIC distributes incoming packets into queues. The NIC inspects the L4 headers
(i.e. TCP or UDP) and uses the src/dst ip/port pairs to determine which queue
to send the packet to. This means that, if a device has one active TCP connection
and the connection sends lots of data, then only one Tx/Rx queue will be utilized,
i.e. all packets related to the connection end up in the same HW queue. In other words,
this one queue should be in NAPI polling mode while the other queues can rely on
interrupts.

For most network devices single-flow bandwidth is mentioned separately and is
practically always less. The above paragraph should explain why this is the case,
i.e. given a single TCP connection, it can only utilize one Rx queue on the device.
This also means that only one CPU is utilized for packet processing inside the
kernel.

### Interrupt Handlers

The Linux kernel knows two types of interrupts: hard and soft. A hard IRQ is a general
hardware interrupt fired by a hardware device. Soft IRQs on the other hand are
purely software constructs within the kernel.

#### Hard IRQs

The driver has two interrupt handlers. One for management events, which for network
devices respond to events like a cable being yanked, and one for I/O events like
packet arrival or packet send completion.

When registering an interrupt handler, we can provide the Linux IRQ handling API
with a pointer to some data, which is always passed to the handler. In the ENA
driver, we pass a pointer to an [ena_napi struct][ena-napi-struct]. Each Rx/Tx queue
pair on the card registers their own interrupt handler, so we get a pointer to the
`ena_napi` struct of that particular queue.

Interrupt handlers need to be extremely small and quick to run as an interrupt might
fire when the CPU is doing something important. As explained in the section on NAPI
the only thing it does is turn on NAPI polling:

```c
static irqreturn_t ena_intr_msix_io(int irq, void *data)
{
	struct ena_napi *ena_napi = data;

	/* Used to check HW health */
	WRITE_ONCE(ena_napi->first_interrupt, true);

	WRITE_ONCE(ena_napi->interrupts_masked, true);
	smp_wmb(); /* write interrupts_masked before calling napi */

	/* Turn on polling in NAPI */
	napi_schedule_irqoff(&ena_napi->napi);

	return IRQ_HANDLED;
}
```
[I/O interrupt handler][ena-io-interrupt] with additional comments.

To further trace the core NAPI code we have:

```c
void __napi_schedule_irqoff(struct napi_struct *n)
{
	if (!IS_ENABLED(CONFIG_PREEMPT_RT))
		____napi_schedule(this_cpu_ptr(&softnet_data), n);
	else
		__napi_schedule(n);
}

/* Called with irq disabled */
static inline void ____napi_schedule(struct softnet_data *sd,
				     struct napi_struct *napi)
{
	list_add_tail(&napi->poll_list, &sd->poll_list);
	__raise_softirq_irqoff(NET_RX_SOFTIRQ);
}
```
[NAPI poll scheduling code][napi-schedule]

We see that `napi_schedule()` just schedules a `NET_RX_SOFTIRQ`. We look more
closely at this in the next subsection on soft interrupts.

Finally, we also have the management interrupt handler, which is not very
interesting from a driver user's perspective. The code is the following:

```c
static irqreturn_t ena_intr_msix_mgmnt(int irq, void *data)
{
	struct ena_adapter *adapter = (struct ena_adapter *)data;

	ena_com_admin_q_comp_intr_handler(adapter->ena_dev);

	/* Don't call the aenq handler before probe is done */
	if (likely(test_bit(ENA_FLAG_DEVICE_RUNNING, &adapter->flags)))
		ena_com_aenq_intr_handler(adapter->ena_dev, data);

	return IRQ_HANDLED;
}
```
[Management interrupt handler][ena-mgmt-interrupt] with additional comments.

### Soft IRQs

Software interrupts (soft IRQs) are a mechanism to signal the kernel that some
code needs to run. They are reserved for the most high-priority tasks in the kernel.
Soft IRQs are simply per-CPU bitvectors and are set as follows:

```c
/* A per-CPU bitvector */
#define local_softirq_pending_ref irq_stat.__softirq_pending
/* Or the bitmask with the per-CPU bitvector */
#define or_softirq_pending(x)	(__this_cpu_or(local_softirq_pending_ref, (x)))

void __raise_softirq_irqoff(unsigned int nr)
{
	lockdep_assert_irqs_disabled();
	trace_softirq_raise(nr);
	or_softirq_pending(1UL << nr);
}
```
[raise softirq()][raise-softirq]

From the code of [napi_schedule()][napi-schedule] in the previous section, we
see that NAPI code sets the `NET_RX_SOFTIRQ` bitfield. All the different soft IRQs
are defined in an [enum][soft-irqs]. `irq_stat` is simply a per CPU instance of the
architecture dependent struct [irq_cpustat_t][irq-stat].

One detail is worth pointing out. Notice that the soft IRQ is
per processor. In particular, since the HW interrupt handler schedules the soft IRQ,
both are executed on the same CPU. This makes sense, so that processing the same data
happens on the same CPU, which improves data locality.

Two questions remain open:

1. What does the software interrupt handler do?
2. When is the software interrupt handler called?

A soft IRQ handler is registered with a call to the function [open_softirq()][open-softirq].
The handler for `NET_RX_SOFTIRQ` is registered in [net_dev_init()][net-dev-init] and
is the following:

```c
static __latent_entropy void net_rx_action(struct softirq_action *h)
{
	...
	for (;;) {
		struct napi_struct *n;

		if (list_empty(&list)) {
			if (!sd_has_rps_ipi_waiting(sd) && list_empty(&repoll))
				goto out;
			break;
		}

		n = list_first_entry(&list, struct napi_struct, poll_list);
		budget -= napi_poll(n, &repoll);

		/* If softirq window is exhausted then punt.
		 * Allow this to run for 2 jiffies since which will allow
		 * an average latency of 1.5/HZ.
		 */
		if (unlikely(budget <= 0 ||
			     time_after_eq(jiffies, time_limit))) {
			sd->time_squeeze++;
			break;
		}
	}
	...
}
```
[snippet from net_rx_action()][net-rx-action]

Remember that [napi_schedule()][napi-schedule] stored the `napi_struct` of the
NIC when it was called. [net_rx_action()][net-rx-action] then loops over all the
stored `napi_poll()` handlers and calls them. In other words, here we finally
call [ena_io_poll()][ena-io-poll] in the driver.

What remains is understanding when the soft IRQ handler is called. Currently,
soft IRQs can run in two places:

1. When returning from a hardware interrupt (in `irq_exit()`)
2. By the `ksoftirqs` kernel thread

In practice, when a server is under high load they get mostly executed by the
ksoftirqd.

### Packet Processing

We have now covered what happens when a packet reaches the machine and how it ends
up being processed by the driver. In other words, how [ena_io_poll()][ena-io-poll]
gets called. To summarize, the steps are:

1. HW interrupt handler in the driver fires, i.e. [ena_intr_msix_io()][ena-io-interrupt]
2. HW interrupt handler calls [napi_schedule()][napi-schedule]
3. [napi_schedule()][napi-schedule] calls [raise_softirq()][raise-softirq]
4. [raise_softirq()][raise-softirq] sets `NET_RX_SOFTIRQ` bit in [irq_stat][irq-stat]
5. [ena_io_poll()][ena-io-poll] is called when soft IRQs is handled

Next we look at the how packets are pushed to the next layer of stack, i.e. look at
the implementation of [ena_io_poll()][ena-io-poll]. Finally, we cover how packets are
sent.

#### Incoming Data (Rx)

As mentioned before, the hardware can either fire an interrupt when receiving
a packet or the interrupts can be turned off with packets being received by
periodically polling the driver.

The NAPI framework calls [ena_io_poll][ena-io-poll] when polling the NIC for data,
which at a high-level does the following:

1. Gets the Rx/Tx ring buffers for the queue that scheduled the soft IRQ. 
2. Handles any completed Tx packets.
3. Handles any pending Rx packets up to provided work budget.
4. If the number of handled Rx/Tx packets less than budget, turn interrupts back on.

We next go over the function code (comments added):

```c
static int ena_io_poll(struct napi_struct *napi, int budget)
{
	struct ena_napi *ena_napi = container_of(napi, struct ena_napi, napi);
	struct ena_ring *tx_ring, *rx_ring;
	int tx_work_done;
	int rx_work_done = 0;
	int tx_budget;
	int napi_comp_call = 0;
	int ret;

	/* 
	 * Step 1
	 */
   
	tx_ring = ena_napi->tx_ring;
	rx_ring = ena_napi->rx_ring;

	tx_budget = tx_ring->ring_size / ENA_TX_POLL_BUDGET_DIVIDER;

	/* Test Rx/Tx queues are still up and the watchdog hasn't fired. */
	if (!test_bit(ENA_FLAG_DEV_UP, &tx_ring->adapter->flags) ||
	    test_bit(ENA_FLAG_TRIGGER_RESET, &tx_ring->adapter->flags)) {
		napi_complete_done(napi, 0);
		return 0;
	}
#ifdef ENA_BUSY_POLL_SUPPORT
	if (!ena_bp_lock_napi(rx_ring))
		return budget;
#endif

	/*
	 * Step 2
	 */

    /* tx_budget does NOT depend on budget provided by napi_poll and is fixed. */
	tx_work_done = ena_clean_tx_irq(tx_ring, tx_budget);

	/*
	 * Step 3
	 */
	 
	/* If called with zero budget, we don't do any Rx handling. */
	if (likely(budget))
		rx_work_done = ena_clean_rx_irq(rx_ring, napi, budget);

	/* Recheck Rx/Tx queues still up and watchdog hasn't fired. */
	if (unlikely(!test_bit(ENA_FLAG_DEV_UP, &tx_ring->adapter->flags) ||
		     test_bit(ENA_FLAG_TRIGGER_RESET, &tx_ring->adapter->flags))) {
		napi_complete_done(napi, 0);
		ret = 0;

	/*
	 * Step 4: If branch taken, interrupts are turned back on.
	 */
	 
	} else if ((budget > rx_work_done) && (tx_budget > tx_work_done)) {
		napi_comp_call = 1;

		/* Update numa and unmask the interrupt only when schedule
		 * from the interrupt context (vs from sk_busy_loop)
		 */
#if LINUX_VERSION_CODE >= KERNEL_VERSION(4, 10, 0)
		if (napi_complete_done(napi, rx_work_done) &&
		    READ_ONCE(ena_napi->interrupts_masked)) {
#else
		napi_complete_done(napi, rx_work_done);
		if (READ_ONCE(ena_napi->interrupts_masked)) {
#endif
			smp_rmb(); /* make sure interrupts_masked is read */
			WRITE_ONCE(ena_napi->interrupts_masked, false);
			/* We apply adaptive moderation on Rx path only.
			 * Tx uses static interrupt moderation.
			 */
			if (ena_com_get_adaptive_moderation_enabled(rx_ring->ena_dev))
				ena_adjust_adaptive_rx_intr_moderation(ena_napi);

			ena_update_ring_numa_node(tx_ring, rx_ring);
			ena_unmask_interrupt(tx_ring, rx_ring);
		}

		ret = rx_work_done;
	} else {
		ret = budget;
	}

	u64_stats_update_begin(&tx_ring->syncp);
	tx_ring->tx_stats.napi_comp += napi_comp_call;
	tx_ring->tx_stats.tx_poll++;
	u64_stats_update_end(&tx_ring->syncp);

#ifdef ENA_BUSY_POLL_SUPPORT
	ena_bp_unlock_napi(rx_ring);
#endif
	tx_ring->tx_stats.last_napi_jiffies = jiffies;

	return ret;
}
```
[ena_io_poll() with comments][ena-io-poll]

In step (3) above, the actual work is done by the function [ena_clean_rx_irq()][ena-clean-rx].
This function unmaps the packet buffers from DMA and creates a [sk_buff][skbuff] (socket buffer) object
and sends the data to the next layer. I recommend the following [video][skbuff-video] for an overview
of what sk_buffs are.

The function [ena_clean_rx_irq()][ena-clean-rx] basically does the following for each buffer in the
Rx queue's ring buffer:

1. Unmaps the buffer from DMA
2. Wraps the buffer in an [sk_buff][skbuff].
3. Sends packet to next layer using either [netif_receive_skb()][netif_receive_skb] or [napi_gro_receive()][napi_gro_receive].

Note that ownership of the buffers onto which packets are written to by DMA is transferred
to the `sk_buff`. The upper layers of the stack free the memory once they are done. The
advantage is that we don't need to copy the packet.

#### Outgoing Data (Tx)

TODO

## Driver Layer Configuration

With some understanding of how the driver works we now take a look at some configuration
options available.

TODO

## Inspecting the Driver with eBPF

Demonstrate:

1. `ifconfig ena1 up` and `ifconfig ena1 down`
2. Call to `ena_io_poll()`
3. Tx queue submission

## Why Is This Useful?

Explain what kind of software can be accelerated by understanding how the kernel works.

TODO: Figure out the following:

- static interrupt delay
- interrupt moderation (can be adjusted with `ethtool -c`)
- driver busy polling

[ena-adapter-struct]: https://github.com/amzn/amzn-drivers/blob/ena_linux_2.8.0/kernel/linux/ena/ena_netdev.h#L370
[ena-napi-struct]: https://github.com/amzn/amzn-drivers/blob/ena_linux_2.8.0/kernel/linux/ena/ena_netdev.h#L139
[ena-io-interrupt]: https://github.com/amzn/amzn-drivers/blob/ena_linux_2.8.0/kernel/linux/ena/ena_netdev.c#L1742
[ena-mgmt-interrupt]: https://github.com/amzn/amzn-drivers/blob/ena_linux_2.8.0/kernel/linux/ena/ena_netdev.c#L1725
[ena-init]: https://github.com/amzn/amzn-drivers/blob/ena_linux_2.8.0/kernel/linux/ena/ena_netdev.c#L4855
[ena-open]: https://github.com/amzn/amzn-drivers/blob/ena_linux_2.8.0/kernel/linux/ena/ena_netdev.c#L2552
[ena-probe]: https://github.com/amzn/amzn-drivers/blob/ena_linux_2.8.0/kernel/linux/ena/ena_netdev.c#L4418

[netdev]: https://github.com/amazonlinux/linux/blob/kernel-5.10.147-133.644.amzn2/include/linux/netdevice.h#L1879
[register-netdev]: https://github.com/amazonlinux/linux/blob/kernel-5.10.147-133.644.amzn2/net/core/dev.c#L9910
[napi-schedule]: https://github.com/amazonlinux/linux/blob/kernel-5.10.147-133.644.amzn2/net/core/dev.c#L4272
[napi-complete]: https://github.com/amazonlinux/linux/blob/kernel-5.10.147-133.644.amzn2/net/core/dev.c#L6482
[napi-poll]: https://github.com/amazonlinux/linux/blob/kernel-5.10.147-133.644.amzn2/net/core/dev.c#L6813
[ena-io-poll]: https://github.com/amzn/amzn-drivers/blob/ena_linux_2.8.0/kernel/linux/ena/ena_netdev.c#L1642
[ena-clean-rx]: https://github.com/amzn/amzn-drivers/blob/ena_linux_2.8.0/kernel/linux/ena/ena_netdev.c#L1371
[raise-softirq]: https://github.com/amazonlinux/linux/blob/kernel-5.10.147-133.644.amzn2/kernel/softirq.c#L482
[irq-stat]: https://github.com/amazonlinux/linux/blob/kernel-5.10.147-133.644.amzn2/arch/x86/include/asm/hardirq.h#L7
[open-softirq]: https://github.com/amazonlinux/linux/blob/kernel-5.10.147-133.644.amzn2/kernel/softirq.c#L489
[net-dev-init]: https://github.com/amazonlinux/linux/blob/kernel-5.10.147-133.644.amzn2/net/core/dev.c#L11266
[net-rx-action]: https://github.com/amazonlinux/linux/blob/kernel-5.10.147-133.644.amzn2/net/core/dev.c#L6879
[soft-irqs]: https://github.com/amazonlinux/linux/blob/kernel-5.10.147-133.644.amzn2/include/linux/interrupt.h#L528
[skbuff-video]: https://youtu.be/6Fl1rsxk4JQ?t=488
[skbuff]: https://github.com/amazonlinux/linux/blob/kernel-5.10.147-133.644.amzn2/include/linux/skbuff.h#L714
[netif_receive_skb]: https://github.com/amazonlinux/linux/blob/kernel-5.10.147-133.644.amzn2/net/core/dev.c#L5633
[napi_gro_receive]: https://github.com/amazonlinux/linux/blob/kernel-5.10.147-133.644.amzn2/net/core/dev.c#L6146

[bpf-reuseport]: https://elixir.bootlin.com/linux/latest/source/tools/testing/selftests/net/reuseport_bpf_cpu.c
[checksum-offload]: https://www.kernel.org/doc/html/latest/networking/checksum-offloads.html
[IP-checksum]: https://en.wikipedia.org/wiki/Internet_checksum
[TSO]: https://www.kernel.org/doc/html/latest/networking/segmentation-offloads.html
[RSS]: https://www.kernel.org/doc/html/latest/networking/scaling.html
[iperf3]: https://iperf.fr/
[test-env]: https://github.com/efagerho/linux-networking-test-env
[flamegraph]: https://github.com/brendangregg/FlameGraph
[linux-kernel]: https://github.com/amazonlinux/linux/tree/kernel-5.10.147-133.644.amzn2
[ENA-driver]: https://github.com/amzn/amzn-drivers/tree/ena_linux_2.8.0/kernel/linux/ena
