# The Pauseless GC Algorithm

# Pauseless GC �㷨���� - ���ķ���

> ##### ����ע: 
> Pauseless GC Algorithm, ��ͣ�ٵ������ռ��㷨; �㷨��������STWͣ�ٵģ������� HotSpot JVM ��ʵ�ֻ���������STWͣ�١�
> mutator, ҵ���̣߳����ڴ���ж�д���߳�, �����Ϊҵ���߳�, ��GC�߳������֡�

```
����: 
Cliff Click ;  Gil Tene ; Michael Wolf ; {cliffc,gil,wolf}@azulsystems.com

Azul Systems, Inc.
1600 Plymouth Street
Mountain View, CA 94043
```

> Permission to make digital or hard copies of all or part of this work for personal or classroom use is granted without fee provided that copies are not made or distributed for profit or commercial advantage and that copies bear this notice and the full citation on the first page. To copy otherwise, or republish, to post on servers or to redistribute to lists, requires prior specific permission and/or a fee.

> �Ը���ѧϰ���ѧ�о�ΪĿ��, �������ʹ�ñ����ģ�����������ıȡ�������ҵ���档 ��Ҫ�ڵ�һҳ��չʾ����������������ʽ�������������������ĵ�������Ҫ������/���ѡ�

> VEE��05, June 11�C12, 2005, Chicago, Illinois, USA.

> Copyright 2005 ACM 1-59593-047-7/05/0006...$5.00.


## ABSTRACT

## ժҪ

Modern transactional response-time sensitive applications have run into practical limits on the size of garbage collected heaps. The heap can only grow until GC pauses exceed the response-time limits. Sustainable, scalable concurrent collection has become a feature worth paying for.

����кܶ�ϵͳ��ҵ�����Ӧʱ��ǳ�����, ȴ��GC����ܵ����ڴ��С�����ơ� ���ǵĶ��ڴ治�����õ÷ǳ��󣬱������ƴ�С����GC��ͣʱ������ҵ������������Ӧʱ�䡣 ҵ��������Ҫһ����Գ����ȶ����еġ����ݸ����ڴ��ģ�Ĳ��������ռ�����

Azul Systems has built a custom system (CPU, chip, board, and OS) specifically to run garbage collected virtual machines. The custom CPU includes a read barrier instruction. The read barrier enables a highly concurrent (no stop-the-world phases), parallel and compacting GC algorithm. The Pauseless algorithm is designed for uninterrupted application execution and consistent mutator throughput in every GC phase.

Azul Systems ��˾ר�Ŷ�����һ��ϵͳ(����CPU��оƬ�顢���弰����ϵͳ)������֧����������������ռ��㷨�� ���Ƶ�CPU֧�֡�������ָ���������(read barrier)��֧�ָ߲���(��ͣ��)������ִ�еġ������ڴ���Ƭ�����ܵ�GC�㷨�� Pauseless GC �㷨����ר����ƣ� ֧�ֲ���ϵ�Ӧ��ִ��, �ڸ���GC�׶ζ��ܱ����ȶ���ҵ����������

Beyond the basic requirement of collecting faster than the allocation rate, the Pauseless collector is never in a ��rush�� to complete any GC phase. No phase places an undue burden on the mutators nor do phases race to complete before the mutators produce more work. Portions of the Pauseless algorithm also feature a ��self-healing�� behavior which limits mutator overhead and reduces mutator sensitivity to the current GC state.

�����������Ҫ��( �����ռ����ٶ�Ҫ����ҵ���ڴ������ٶȣ��� Pauseless �ռ����Ӳ��������ĳ��GC�׶Ρ��κν׶ζ������ҵ���������ĸ�����Ҳ����Ҫ����CPU��Ѹ�����ĳ���׶Ρ� Pauseless�����ռ��㷨�е�ĳЩ���ֻ����С����޸������ܣ�������Ϊ������ҵ���̵߳Ŀ�������������ҵ���̶߳Ե�ǰGC״̬�����жȡ�

We present the Pauseless GC algorithm, the supporting hardware features that enable it, and data on the overhead, efficiency, and pause times when running a sustained workload.

������Ҫ����Pauseless GC�㷨��������ҪӲ��֧�ֵ�����, �Լ���ҵ�񱥺ͳ����µ�GC�������������������ͣʱ������ݡ�


## Categories and Subject Descriptors

## ��������������

D.3.4 [Processors] �C Memory management, 

D.3.3 [Language Constructs and Features] �C Dynamic storage management,

## General Terms

## һ������

Languages, Performance, Design, Algorithms.

## Keywords

## �ؼ���

Read barriers, memory management, garbage collection, concurrent GC, Java, custom hardware

�����ϡ��ڴ���������ռ�������GC��Java��Ӳ������

## 1. INTRODUCTION

## 1. ���

Many of today's enterprise applications are based on garbage collected virtual machine environments such as Java and .NET.

������кܶ���ҵӦ�û������������������, ����Java��.netӦ��, ��ȻҲ����������ṩ���Զ��������ջ��ơ�

Most have response time sensitive components �C for example, a person may be waiting for a web page to load, or a credit-card swipe needs to complete. Stopping for an inopportune GC pause can lead to unacceptable response times. For these applications it is unacceptable for collectors to drive high average throughput numbers at the expense of occasional poor response times.

��һ����ϵͳ����Ӧʱ��ǳ����� ���� ����, ���ڵȴ���ҳ���ص��û�, �����ڵȴ����ÿ�ˢ����ɵ�������.һ�β���ʱ�˵�GC��ͣ���ܵ��²��ɽ��ܵ���Ӧʱ��. ���ⲿ��ϵͳ��˵, ���������ƽ����������ȴӰ�쵽��Ӧʱ�䣬������ȫ���ܽ��ܵġ�

These enterprise applications need a low-pause time collector (pauses on the order of human reflexes, 10-100ms) that can handle very large Java programs (heap sizes from 100MB to 100GB) and highly concurrent workloads (100s of concurrent mutator threads). Such a collector needs to perform consistently and predictably over long periods of time, rather than simply excel at short time-bursts of workload.

������ҵӦ�ó�����Ҫ�ǳ��͵�GC��ͣʱ��(���������ķ�Ӧ, ��Ҫ������10-100ms����), ͬʱ��Ӧ�Էǳ��Ӵ��Java����(���ڴ��СΪ 100MB��100GB�����Χ), �Լ��߲�������(�ϰٸ������޸��߳�).

Many modern garbage collectors rely on write barriers imposed on mutator heap writes, to keep track of references between different heap regions. This enables an efficient generational or region-based GC and is widely used in many garbage-collected languages including most production Java implementations. Read barriers, on the other hand, are rarely used in production systems despite a wealth of academic research because of the high mutator cost they usually incur.

��ǰ����Ƚ��������ռ���, �������ڶ�ҵ���߳�ǿ������д����(write barrier),���ƶ��ڴ�д��, �����ٿ���ͬ����֮�������. ���ڷִ�/������GC�㷨���зǳ���Խ������, �㷺Ӧ���ڸ��ֲ�Ʒ�������ʵ��, ����JVM. ���仰˵�������������к���ʹ�ö�����, �����кܶ��ѧ���о�, ��Ҫ������Ϊҵ���̵߳ĳɱ�̫�ߡ�

Azul Systems has built a custom system (CPU, chip, board, and OS) specifically to run garbage collected virtual machines. The custom CPU includes a read barrier instruction. The read barrier enables a highly concurrent, parallel and compacting GC algorithm. The Pauseless GC algorithm is simple, efficient (low mutator overhead), and has no Stop-The-World pauses.

Azul Systems ��˾������һ��ר�Ŷ��Ƶ�ϵͳ(����CPU��оƬ�顢�����Լ�����ϵͳ)����������֧�������ռ�������������Ƶ�CPU֧��[������ָ��]��������(read barrier)����֧�ָ߲��������л��������ڴ���Ƭ�����ܵ�GC�㷨����ͣ�ٵ�GC�㷨�ǳ���ࡢ��Ч(ͻ�俪����С), û��STWͣ��(Stop-The-World pauses)��



## 2. RELATED WORK

## 2. �������

The idea of garbage collection has been around for a long time [22][13][16][11]. We do not attempt to summarize all relevant GC work and instead we refer the reader to several GC surveys [30][31], and highlight a few papers.

�����ռ������뷨�Ѿ���ܳ���ʱ����(��[22][13][16][11]). ���Ĳ����ǻ������е�GC����, ֻ�ǲο����еĲ���GC�о�(��[30][31]),������ǿ��һ������Ҫ�����ݡ�

GC pauses and their unpredictable impact on mutators was the driving force behind the early work on concurrent collectors [26][5]. The expectation of the time was that special GC hardware would shortly be feasible and commonplace. This early work required such extensive fine-grained synchronization that it would only be feasible on dedicated hardware. GC hardware continues to be proposed to this day [23][29][24][18][20].

GC��ͣ�����ҵ�����ܵ�Ӱ����в���Ԥ����, �ƶ��˲���GC�����ڷ�չ(��[26][5]). ͨ�������Ӳ���豸��֧�ſ�Ԥ�ڵĶ��ݵ�GCʱ��, ��֤���ǿ��еġ��Ǻ��ձ�Ľ������.  ��Щ���ڲ�Ʒʹ���˺ܶ�ϸ���ȵ�ͬ��, ����Ҫ��ר�ŵ�Ӳ��֧�֡���ʱ��GCר�õ�Ӳ������������(��[23][29][24][18][20])��

The idea of using common page-protection hardware to support GC has also been around awhile [2]. Both Appel [2][3] and Ossia [25] protect pages that may contain objects with non-forwarded pointers (initially all pages). Accessing a protected page causes an OS trap which the GC handles by forwarding all pointers on that page, then clearing the page protection. Appel does the forwarding in kernel-mode, Ossia maps the physical memory with a second unprotected virtual address for use by GC. Our Pauseless collector protects pages that contain objects being moved, instead of protecting pages that contain pointers to moved objects. This is a much smaller page set, and the pages can be incrementally protected. Our read-barrier allows us to intercept and correct individual stale references, and avoids blocking the mutator to fix up entire pages. We also support a special GC protection mode to allow fast, non-kernel-mode trap handlers that can access protected pages.

ʹ��ͨ��ҳ�����豸��֧��GC���뷨Ҳ����ʱ�����(��[2]). Appel��˾(��[2][3])��Ossia��˾(��[25])ʹ�÷�ǰ��ָ��(non-forwarded pointers)���������ܰ���������ڴ�ҳ(��ʼ������ҳ��). �������һ���ܱ�����ҳ��, ����ò���ϵͳʹ������������GCָ��, ����ҳ���ϵ�����ָ�����ת��, Ȼ����ȥ��ҳ�汣��.  Appel��˾���ں�ģʽִ��ָ��ת��, Ossia��˾���ǽ������ڴ�ӳ�䵽һ����GCʹ�õĲ��ܱ����������ַ.  �����ǹ�˾ʵ�ֵ�Pauseless�����ռ����ᱣ�������ƶ������ҳ��,������ȥ����ָ���ƶ��������Щҳ��. ���ִ���ʽ�� page set ��С�ö࣬���Ҹ���ҳ������������ܵ�����. ʹ�ö����Ͽ������������ز�����������Ĺ�������, �����޸�����ҳ�棬��������ҵ���߳�. ����Ҳ֧��һ�������GC����ģʽ, ��ҵ���̷߳����ܱ�����ҳ��ʱ, �Է��ں�ģʽ���ÿ������崦�����

-----------

The idea of an incremental collector (via reference counting) is not new either [15]. Incremental collection seeks to reduce pause time by spreading the collection work out in time, finely interleaving GC work with mutator work. Because reference counting is expensive, and indeed all barriers (reference counting typically involves a write barrier) impose some mutator cost there is considerable research in reducing barrier costs [6][21][8] [9]. Having the read-barrier implemented in hardware greatly reduces costs. In our case the typical cost is roughly that of a single cycle ALU instruction.

�����ռ���(ͨ�����ü���)���뷨Ҳ�����������(��[15]). �����ռ���Ŀ����Ϊ�˼���ͣ��ʱ��, ͨ��ʵʱ��GC����, ��ϸ����GC������ҵ���̵߳�ִ��. ��Ϊ���ü����Ĵ��ۺܸ�, ��ʵ�����е�����(���ü���ͨ���漰д�ϰ�)����ǿ��һЩ�޸ĳɱ�, �кܶ��о������ڽ������Ͽ���(��[6][21][8][9]). ��Ӳ����ʵ�ֵĶ����ϴ�󽵵��˿����������ǵ�ʵ����, һ��ֻ��Ҫ����ALUָ���ʱ�����ڡ�

Incremental and low-pause-time collectors are becoming very popular again �C partly because embedded devices have grown in compute power to the point where it's feasible to run a garbage collected language on them [19]. Metronome is an example of a modern low-pause time collector for a uniprocessor embedded system, and the pause times reported for Metronome are indeed smaller than those reported here [4]. However, Metronome as currently described is single-threaded and large business-class applications have enough mutators to overwhelm any singlethreaded collector. Pauseless is fully parallel and can add GC worker threads at any time. Metronome requires an oracle to predict the future GC needs of running applications; this oracle is easily supplied in embedded systems with a fixed application set (the engineer runs the finite application set and measures GC consumption). Servers typically do not have a fixed application set and GC requirements are highly unpredictable. Metronome mutator utilization is around 50%. In contrast our mutator utilization is closer to 98%, we use extra CPUs to do the collection work. In exchange, Metronome provides hard real-time guarantees while we provide only soft real-time guarantees.

�����͵�ͣ��ʱ��������ռ����ٴα���������� ���� ����ԭ����Ƕ��ʽ�豸����������Ѹ������, �Ѿ��������������������ռ�������(��[19]).Metronome��һ�������ڵ�������Ƕ��ʽ�豸�е�GC, ������ͣʱ��ݱ���ȷʵ��������GCҪ��(��[4]). ������, Metronome Ŀǰ����һ��̵߳������ռ���, �����͵���ҵӦ�û��зǳ����ҵ���߳�, �����׾���ѹ�����е��̵߳�GC. ��ͣ��GC����ȫ���е�,�������κ�ʱ������GC�����߳�. Metronome��Ҫһ��������(oracle)��Ԥ��δ��GC���е�����; ��Ƕ��ʽϵͳ����������ߺ������ṩ����Ϊ��Ҫ���еĳ�������Ͼ���Щ(����ʦ��д��Ӧ�ó��������޵�,���ҿ��Բ�ȡһЩGC���ĵĴ�ʩ)��������������Ҫִ�еĳ�����ȥ��, ����GC������������ǲ���Ԥ��ġ�Metronome��ҵ���߳������ʴ�Լ��50%. ���֮�����ǵ�mutator�����ʽӽ�98%, ����ʹ�ö����cpu����������ռ�����. �򵥶Ա�, Metronome�ṩ��Ӳʵʱ��֤, ������ֻ�ṩ��ʵʱ��֤��

Our read-barrier is used for Baker-style relocation [5][23], where the loaded value is corrected before the mutator is allowed to use it. We focus collection efforts on regions which are known to be mostly dead, similar to Garbage-First [14]. Our mark phase uses an incremental update style instead of Snapshot-At-The-Beginning (SATB) style [30]. SATB requires a modestly expensive write-barrier which first does a read (and generally a series of dependent tests). The Pauseless collector does not require a write barrier.

���ǽ����������� Baker-style ���ض�λ[5][23], ��ҵ���߳�ʹ��֮ǰ, ����ֵ�ᱻ����. ���ǽ��ռ���Ŀ�꼯������Щ�󲿷ֶ���������������, ��һ��������G1(��[14]), �ڱ�ǽ׶�ʹ���������µķ�ʽ, �����ǿ��շ�ʽ(Snapshot-At-The-Beginning,SATB),��[30]. SATB��Ҫʹ��һЩ�����д����, ����Ҫ��ȡ(ͨ������һϵ�е���ز���)��Pauseless �ռ�������Ҫʹ��д���ϡ�

Concurrent GCs are available in most modern production JVMs; BEA's JRockit [7], SUN's HotSpot [28] and IBM's production JVM [27] all have concurrent collectors and we tested with the latest available versions of Java 1.4 from each vendor. However, in all cases these collectors are not the defaults. They appear to not be as stable as the parallel collectors and they sometimes put high overheads on mutator threads. For some of these collectors, worse-case transaction times were no better than the default collectors.

����GC�����Ĵ󲿷�������JVM�ж�����ʹ��; BEA��JRockit(��[7]), SUN��˾��HotSpot(��[28])���Լ�IBM�� production JVM([27]) ���Դ���concurrent�ռ���, ���ǲ����˸��ҹ�˾�ṩ�� Java 1.4�汾�������ռ���. Ȼ��,��Щ�ռ���������Ĭ���ռ����������ƺ�û�в��������ռ����ȶ�, ��ʱ��ᵼ��mutator�̷߳ǳ��ߵĿ���. ��Щ�ռ���, ��������, �������Ӧ������Ĭ�ϵ��ռ����á�



## 3. HARDWARE SUPPORT

## 3. Ӳ��֧��

### 3.1 Background

### 3.1 ����

Azul Systems has built a custom system (CPU, chip, board, and OS) specifically to run garbage collected virtual machines such as Java; the JVM is based on SUN's HotSpot [28]. We describe actual production hardware, which had real costs to design, develop and debug. Thus we were strongly motivated to design simple and cost-effective hardware. In the end, the custom GC hardware we built was quite minor.

Azul Systems ��˾ר�Ź�����һ�׶���ϵͳ��CPU��оƬ�飬��·��Ͳ���ϵͳ��������������Java����֧�������ռ��������; ����JVM����SUN��˾��HotSpot([28])������������ʵ�ʵ�������Ӳ��������ƣ������͵��Թ����ж�������ܶ�ʵ�ʵĳɱ�����ˣ����Ƿǳ���Ҫ��Ƴ�һ�ּȼ���Ҫ���Լ۱ȵ�Ӳ���豸�����յĽ���ǣ����ǹ��������Ƴ���һ��ǳ����ɵ�GCӲ����

The basic CPU core is a 64-bit RISC optimized to run modern managed languages like Java. It makes an excellent JIT target but does not directly execute Java bytecodes. Each chip contains 24 CPUs, and up to 16 such chips can be made cache-coherent; the maximum sized system has 384 CPU cores and 256G of memory in a flat, symmetric memory space. The box runs a custom OS and can run as many JVMs as memory allows. A single JVM can dynamically scale to use all CPUs and all available memory.

ʹ�þ����Ż���64λRISC�ܹ�CPU���ģ���������������Java�������ִ��й����ԡ����Ա����һ��׿Խ��JITĿ����룬����ֱ��ִ��Java�ֽ��롣ÿ��оƬ�����24��CPU��ʵ�ֻ���һ�µ�ǰ����, ����������16��������оƬ�鼶��; ����ϵͳ���������Ϊ384��CPU�ں�, 256G�ڴ�ĶԳ��ڴ�ռ䡣����ϵͳ���ж��ƵĲ���ϵͳ��ֻҪ�ڴ���������ͬʱ���кܶ��JVM������JVM���Ը�����Ҫ��ʹ�����е�CPU�Ϳ����ڴ档

The hardware supports a number of fast user-mode trap handlers. These trap handlers can be entered and left in a handful of clock cycles (4-10, depending) and are frequently used by the GC algorithm; fast traps are key. The hardware also supports a fast cooperative preemption mechanism via interrupts that are taken only on user-selected instructions.

Ӳ��֧�ֶ�������û�ģʽ�����崦��������Щ���崦���������ڽ������һ����ʱ�����ڣ�4-10���������һᾭ����GC�㷨ʹ��; �������������еĹؼ���ͨ�������û�ѡ���ָ���ж�, ����Ӳ����֧�ֿ���Э����ռ���ơ�

### 3.2 OS-level Support

### 3.2 ����ϵͳ������֧��

The hardware TLB supports an additional privilege level, the GC-mode, between the usual user- and kernel-modes. Usage of this GC-mode is explained in the GC algorithm section. Several of the fast user-mode traps start the trap handler in GC-mode instead of user-mode. The TLB also supports 1 megabyte pages; the 1M page thus becomes the standard unit of work for the Pauseless GC algorithm and appears frequently below.

Ӳ��TLB֧�ֶ����Ȩ�޼���, ���û�ģʽ���ں�ģʽ֮�����һ��GCģʽ(GC-mode)��������GCģʽ����GC�㷨���֡���һЩ�����û�ģʽ���彫����GCģʽ���������崦�����,�������û�ģʽ��TLB��֧��1MB���ڴ�ҳ; ����1MB���ڴ�ҳ��Pauseless GC�㷨�ı�׼������Ԫ�����Ļᾭ���ᵽ��


The TLB is managed by the OS in the usual ways, with normal kernel-level TLB trap handlers being invoked when normal loads and stores fail an address translation. Setting the GC privilege mode bit is done by the JVM via calls into the OS. TLB violations on GC-protected pages generate fast user-level traps instead of OS level exceptions.

TLB�ɲ���ϵͳ�Գ��淽ʽ�����������ļ��غʹ洢ʧ�ܵ�����Ҫ���е�ַת��ʱ�������������ں˼�TLB���崦�����GCȨ��ģʽ��־λ��������JVM���ò���ϵͳAPI����ɡ���GC�������ڴ�ҳ�ϵ�Υ��TLB����, �����ɿ����û������壬�����ǲ���ϵͳ���쳣��

HotSpot supports a notion of GC safepoints, code locations where we have precise knowledge about register and stack locations [1]. The hardware supports a fast cooperative preemption mechanism via interrupts that are taken only on user-selected instructions, allowing us to rapidly stop individual threads only at safepoints. Variants of some common instructions (e.g., backwards branches, function entries) are flagged as safepoints and will check for a pending per-CPU safepoint interrupt. If a safepoint interrupt is pending the CPU will take an exception and the OS will call into a user-mode safepoint-trap handler. The running thread, being at a known safepoint, will then save its state in some convenient format and call the OS to yield. When the OS wants to preempt a normal Java thread, it sets this bit and briefly waits for the thread to yield. If the thread doesn't report back in a timely fashion it gets preempted as normal.

HotSpot֧��GC��ȫ��(safepoint)�ĸ��Ҳ���ǶԼĴ����Ͷ�ջλ������ȷ�˽�Ĵ���λ��(��[1])��Ӳ��֧�ֿ���Э����ռ����, ֻ��Ҫ���û�ѡ���ָ����ִ���жϣ���������ֻ�ڰ�ȫ��λ�ÿ���ֹͣĳ���̡߳�һЩ������ָ����磬����֧��������ڣ������Ϊ��ȫ�㣬�������ÿ��CPU�Ĵ�����İ�ȫ���жϡ������ȫ���жϹ���CPU�������쳣������ϵͳ�ͻ�����û�ģʽ�İ�ȫ�����崦������������е�, ������֪��ȫ����߳�, ����ĳ�ַ���ĸ�ʽ������״̬, �����ò���ϵͳ��yield��������ϵͳ��Ҫȡ��������Java�߳�ʱ���������ô˱�־λ, ����ʱ�ȴ��߳� yield������߳�û�м�ʱ���棬������������ռ��

The result of this behavior is that nearly all stopped threads are at GC safepoints already. Achieving a global safepoint, a Stop- The-World (STW) pause, is much faster than patch-and-roll-forward schemes [1] and is without the runtime cost normally associated with software polling schemes. While the algorithm we present has no STW pauses, our current implementation does. Hence it's still useful to have a fast stopping mechanism.

������Ϊ�Ľ��, �Ǽ���������ֹͣ���̶߳��Ѵ���GC��ȫ�㡣ʵ��ȫ�ֵİ�ȫ�㣬Stop-The-World��STW����ͣ���Ȳ�����ǰ������Ҫ��ö�(��[1])������ͨ��û�������ѯ�������������ʱ��������Ȼ����������㷨������û��STW��ͣ������ǰ��ʵ��ȴ����ڡ���ˣ�ӵ�п���ֹͣ������Ȼ�ǳ����á�

We also make use of Checkpoints, points where we cannot proceed until all mutator threads have performed some action. In a Checkpoint each mutator reaches a GC safepoint, does a small amount of GC-related work and then carries on. Blocked threads are already at GC safepoints; GC threads perform the action on their behalf. In a STW pause, all mutators must reach a GC safepoint before any of them can proceed; the pause time is governed by the slowest thread. In a Checkpoint, running threads are never idled and the GC work is spread out in time. The same hardware and OS support is used for both STW pauses and Checkpoints.

���ǻ�ʹ���˼���(Checkpoint)����������ʱ,�����޷�����ִ��, ��Ҫ�ȴ����е�mutator�߳�ִ����ĳЩ�������ܼ������ڼ����У�ÿ��mutator���ᵽ��GC��ȫ�㣬ִ������GC��صĹ�����Ȼ�����ִ�С����������̶߳��Ѿ�����GC��ȫ��; GC�̴߳�����Щ�߳�ִ�в�������STW��ͣ�У���������е�mutator������GC��ȫ����ܼ���ִ��;��ͣʱ�����������߳̾������ڼ����У����е��߳���Զ������У�����GC�����ἰʱ��ɢ��STW��ͣ�ͼ���ʹ����ͬ��Ӳ���Ͳ���ϵͳ��

### 3.3 Hardware Read Barrier

### 3.3 Ӳ��ʵ�ֶ�����

In addition to the standard RISC load/store instruction set, the CPUs have a few custom instructions to aid in object allocation and collection. In this paper we focus on the hardware read barrier. It is instructive to note that this barrier strongly resemble those from 20 years ago [23].

���˱�׼RISC�� load/storeָ�֮�⣬CPU��֧��һЩ�Զ���ָ��, �Ը������������ռ����ڱ����У����ǽ��ص����Ӳ�������ϡ�ֵ��һ����ǣ����ֶ����Ϻ�20��ǰ�ļ�Ϊ����(��[23])��

The read barrier performs a number of checks and is used in different ways during different GC phases. Its behavior is described briefly here, and then again in greater depth in the context of the GC algorithm in the next section. The read barrier is issued after a load instruction and executes in 1 clock. There is a standard load-use penalty which the compiler attempts to schedule around.

������ִ��һϵ�еļ�飬�ڲ�ͬ��GC�׶ζ��Բ�ͬ�ķ�ʽʹ�á����ڼ�Ҫ����������Ϊ����һ�ڻ������GC�㷨��������������ؽ���������Ϊ����������loadָ��֮�󷢳�������1��ʱ��������ִ�С��������ڳ��԰���ʱ����ڱ�׼�� load-use �ͷ���

The read barrier ��looks like�� a standard load instruction, in that it has a base register, an offset and a value register. The base and offset are not used by the barrier checks but are presented to the trap handler and are used in ��self healing��. The value in the value register is assumed to be a freshly loaded ref, a heap pointer, and is cycled through the TLB just like a base address would be. If the ref refers to a GC-protected page a fast usermode trap handler is invoked, hereafter called the GC-trap. The read barrier ignores null refs. Unlike a Brooks-style [10] indirection barrier there is no null check, no memory access, no loaduse penalty, no extra word in the object header and no cache footprint. This behavior is used during the concurrent Relocate phase.

�����ϡ���������һ����׼��loadָ���Ϊ�����л�ַ�Ĵ���(base register)��ƫ����(offset)��ֵ�Ĵ���(value register)����ַ�Ĵ�����ƫ�����������ϰ����ʹ�ã������ṩ�����崦������Լ����ڡ������޸�����ֵ�Ĵ����е�ֵ�ٶ�Ϊ�¼��ص����ã�Ҳ���Ƕ��ڴ�ָ�룬���������ַһ��ѭ��ͨ��TLB���������ָ������GC������ҳ�棬����ÿ����û�ģʽ���崦��������ĳ�ΪGC���塣�����Ϻ���null���á���Brooks���ļ�����ϲ�ͬ(��[10])��������û��null��飬û���ڴ���ʣ�û��loaduse�ͷ�������ͷ��Ҳû�ж�����֣�Ҳû�л���ռ�ÿռ䡣����Ϊ�ڲ����ض�λ�׶Σ�concurrent Relocate phase��ʹ�á�

We also steal 1 address bit from the 64-bit address space; the hardware ignores this bit (masks it off). This bit is called the Not-Marked-Through (NMT) bit and is used during the concurrent Marking phase. The hardware maintains a desired value for this bit and will trap to the NMT-trap if the ref has the wrong flavor. Null refs are ignored here as well.

���ǻ���64λ��ַ�ռ��н�ȡ��1��bit; Ӳ����������bit���������ε�������λ��Ϊδ���ͨ��λ��NMT, Not-Marked-Through�����ڲ�����ǽ׶�ʹ�á�Ӳ��ά����bit��ֵ, ������ÿ��������⣬���������ӵ�NMT���塣ͬ������� null ���á�

Note that the read barrier behavior can be emulated on standard hardware at some cost. The GC protection check can be emulated with standard page protection and the read barrier emulated with a dead load instruction. The NMT check can be emulated by double-mapping memory and changing page protections to reflect the expected NMT bit value. However, using the TLB to check ref privileges means that a failure will trigger a kernellevel TLB trap instead of a fast user-mode trap. Turning this into a user-mode trap will generally have some substantial cost and may require altering the OS. Our read barrier instruction will not trap on a null ref, and null refs are quite common. Emulating this on standard hardware will require a conditional test in the barrier code or mapping page 0. This in turn precludes using normal memory operations from doubling as null-pointer checks, a common optimization in modern JVMs.

��ע�⣬�ڱ�׼��Ӳ��ƽ̨�ϣ�Ҳ����ģ���������Ϊ��������һ���Ŀ�����GC����������ʹ�ñ�׼��ҳ����������ģ�⣬��������ʹ�þ�̬����ָ��(dead load instruction)��ģ�⡣NMT����ģ�⣬����ͨ��˫ӳ���ڴ棬���ϸ���ҳ�����Է�ӳԤ�ڵ�NMTλ��ʵ�֡����ǣ�ʹ��TLB�������Ȩ�ޣ�ʧ����ᴥ���ں˼���TLB���壬�����ǿ����û�ģʽ���塣����ת��Ϊ�û�ģʽ����ͨ�������һЩʵ���ԵĿ��������ܻ���Ҫ�޸Ĳ���ϵͳ���롣���ǵĶ�����ָ�����������ã���������ȴ�ܳ������ڱ�׼Ӳ����ģ����һ��, ��Ҫ�����ϴ������0��ӳ��ҳ���Ͻ����������ԡ��ⷴ������ֹ��˫���Ŀ�ָ����, �����ִ�JVM�г������Ż���

## 4. THE PAUSELESS GC ALGORITHM

## 4. Pauseless GC �㷨

The Pauseless GC Algorithm is divided into three main phases: Mark, Relocate and Remap. Each phase is fully parallel and concurrent. Mark bits go stale; objects die over time and the mark bits do not reflect the changes. The Mark phase is responsible for periodically refreshing the mark bits. The Relocate phase uses the most recently available mark bits to find pages with little live data, to relocate and compact those pages and to free the backing physical memory. The Remap phase updates every relocated pointer in the heap.

Pauseless GC�㷨��Ϊ������Ҫ�׶Σ�Mark����ǣ���Relocate(�ض�λ)��Remap(��ӳ��)��ÿ���׶ζ�����ȫ���кͲ����ġ����λ��ó¾�;������ʱ�����������λ����ӳ�������ǽ׶θ�����ˢ�±��λ���ض�λ�׶�ʹ��������õı��λ�����Ҿ��������������ҳ�棬���ض�λ��ѹ����Щҳ��, �ͷ������ڴ档��ӳ��׶θ��¶��е�ÿ���ض�λָ�롣

**There is no ��rush�� to finish any given phase.** No phase places a substantial burden on the mutators that needs to be relieved by ending the phase quickly. There is no ��race�� to finish some phase before collection can begin again �C Relocation runs continuously and can immediately free memory at any point. Since all phases are parallel, GC can keep up with any number of mutator threads simply by adding more GC threads. Unlike other incremental update algorithms, there is no re-Mark or final- Mark phase; the concurrent Mark phase will complete in a single pass despite the mutators busily modifying the heap. GC threads do compete with mutator threads for CPU time. On Azul's hardware there are generally spare CPUs available to do GC work. However, ��at the limit�� some fraction of CPUs will be doing GC and will not be available to the mutators.

**�κ��ض��׶ζ�����Ҫ����æ�����**��û���ĸ��׶���Ҫ������ɣ������˸�ҵ���̴߳����ĳ��ظ������������ռ��ٴο�ʼ֮ǰû�С����š����ĳ���׶� - �ض�λ���������У��������κ�ʱ�������ͷ��ڴ档�������н׶ζ��ǲ��еģ����ֻ����Ӹ���GC�̣߳�GC�Ϳ��Ը�������������mutator�̡߳����������������㷨��ͬ������û�����±�ǻ����ձ�ǽ׶�; ����mutatoræ���޸Ķ��ڴ棬��������ǽ׶ν���һ�δ�������ɡ�GC�߳�ȷʵ��mutator�߳���ǹCPUʱ�䡣��Azul��Ӳ���ϣ�ͨ���б���CPU������GC���������ң����ڼ�������¡�ĳЩCPUִֻ��GC����, ������ҵ���߳�ʹ�á�

**Each of the phases involves a ��self-healing�� aspect,** where the mutators immediately correct the cause of each read barrier trap by updating the ref in memory. This assures the same ref will not trigger another trap. The work involved varies by trap type and is detailed below. Once the mutators' working sets have been handled they can execute at full speed with no more traps. During certain phase shifts mutators suffer through a ��trap storm��, a high volume of traps that amount to a pause smeared out in time. We measured the trap storms using Minimum Mutator Utilization, and they cost around 20ms spread out over a few hundred milliseconds.

**ÿ���׶ζ��漰�������޸���**��mutator �߳�ͨ�������ڴ��е�����, ����������ÿ�����������塣��ȷ����ͬһ�����ò��ᴥ����һ�����塣���漰�Ĺ������������Ͷ��죬���������: һ��������mutator�Ĺ����������ǾͿ���ȫ��ִ�ж����������塣��ĳЩ�׶Σ�ҵ���߳������ˡ�����籩���������������൱��ʵ���ϵ���ͣ������ʹ�� Minimum Mutator Utilization ����������籩���ڼ���ms�ڵĿ�����Լ��20ms��

The algorithm we present has no Stop-The-World (STW) pauses, no places where all threads must be simultaneously stopped. However, for ease of engineering into the existing HotSpot JVM our implementation includes some STWs. We feel these STWs can be readily engineered to have pause times below standard OS context- switch times, where a GC pause will be indistinguishable from being context switched by the OS. We will mention where the implementation differs from theory as the phases are described.

����������㷨û��Stop-The-World��STW����ͣ��û�б���ͬʱֹͣ�����̵߳ĵط������ǣ�Ϊ�������е�HotSpot JVM���ݣ����ǵ�ʵ�ְ�����һЩSTW��������Ϊ��ЩSTW���Ժ����׵����Ϊ���ڱ�׼OS�������л����ĵ���ͣʱ�䣬����GC��ͣ��ʱ�佫��OS���������л�ʱ���𲻴����ǽ�������ʱ�ἰ������ʵ�ֵĲ�֮ͬ����

### 4.1 Mark Phase

### 4.1 Mark Phase(��ǽ׶�)

The Mark phase is a parallel and concurrent incremental update (not SATB) marking algorithm [17], augmented with the read barrier. The Mark phase is responsible for marking all live objects, tagging live objects in some fashion to distinguish them from dead objects. In addition, each ref has it's NMT bit set to the expected value. The Mark phase also gathers per-1M-page liveness totals. These totals give a conservative estimate of live data on a page (hence a guaranteed amount of reclaimable space) and are used in the Relocate phase.

��ǽ׶��ǲ��еģ�����ʹ�õ��ǲ����������±���㷨��������SATB��(��[17])�������˶����ϡ���ǽ׶θ��������д��Ķ�����ĳ�ַ�ʽ���ϱ�ǩ�����������������������ֿ��������⣬��ÿ�����ö����ö�Ӧ��NMTλԤ��ֵ����ǽ׶λ��ռ�ÿ��1Mҳ�Ĵ�������������Щ������ҳ���ϵĴ�����ݽ��б��ع��ƣ��Ա�֤�ɻ��տռ������������ض�λ�׶�ʹ�á�

The basic idea is straightforward: the Marker starts from some root set (generally static global variables and mutator stack contents) and begins marking reachable objects. After marking an object (and setting the NMT bit), the Marker then marksthrough the object �C recursively marking all refs it finds inside the marked object. Extensions to make this algorithm parallel have been previously published [17]. Making marking fully concurrent is a little harder and the issues are described further below.

����˼��ܼ򵥣���ǳ����GC����ͨ���Ǿ�̬ȫ�ֱ�����mutator�̵߳�ջ�ռ䣩��ʼ����ǿɴ�Ķ��󡣱��ĳ�����󣨲�����NMTλ��֮�󣬱�ǳ��������ö��� - �ݹ�ر�Ǹö����е��������á�ʹ���㷨����ִ�е���չ�Ѿ�����(��[17])��ʹ�����ȫ�����е����ѣ����潫��һ��������Щ���⡣

### 4.2 Relocate Phase

### 4.2 Relocate Phase(�ض�λ�׶�)

The Relocate phase is where objects are relocated and pages are reclaimed. A page with mostly dead objects is made wholly unused by relocating the remaining live objects to other pages. The Relocate phase starts by selecting a set of pages that are above a given threshold of sparseness. Each page in this set is protected from mutator access, and then live objects are copied out. Forwarding information tracking the location of relocated objects is maintained outside the page.

�ض�λ�׶ν��ж������¶�λ�������ڴ�ҳ�����ĳ��ҳ���д󲿷���������������Խ�ʣ�µĴ��������¶�λ������ҳ�棬��ҳ������ȫ��ʹ�á��ض�λ�׶�����ѡ��һ��ϡ��ֵ(sparseness)���ڸ�����ֵ��ҳ�档��ѡ�е�ÿ��ҳ�涼�ܵ���������ֹҵ���̷߳��ʣ�Ȼ�󽫴����󿽱���ȥ�������ض�λ�����λ��ת����Ϣ��ҳ���ⲿά����

If a mutator loads a reference to a protected page, the read-barrier instruction will trigger a GC-trap. The mutator is never allowed to use the protected-page reference in a language-visible way. The GC-trap handler is responsible for changing the stale protected-page reference to the correctly forwarded reference.

��� mutator ��Ҫ�����ܱ���ҳ���е����ã��������ָ��ᴥ��һ��GC���塣�����Կɼ��Ĳ�����Զ������mutatorʹ���ܱ���ҳ������á�GC���崦��������ܱ���ҳ���й�ʱ���ø���Ϊ��ȷ��ת�����á�

After the page contents have been relocated, the Relocate phase frees the **physical** memory; it's contents are never needed again. The physical memory is recycled by the OS and can immediately be used for new allocations. **Virtual** memory cannot be freed until no more stale references to that page remain in the heap, and that is the job of the Remap phase.

ҳ���е�����ȫ�����¶�λ���ض�λ�׶��ͷ� **�����ڴ�**; Ҳ�������е�������Զ������Ҫ�ˡ������ڴ��ɲ���ϵͳ���գ������������µ��ڴ���䡣**�����ڴ�** ��ַ���������ͷţ�ֱ�����ڴ��в����жԸ�ҳ��Ĺ�ʱ���ã��Ǿ���Remap�׶εĹ����ˡ�

As hinted at in Figure 1, a Relocate phase runs constantly freeing memory to keep pace with the mutators' allocations. Sometimes it runs alone and sometimes concurrent with the next Mark phase.

��ͼ1��ʾ���ض�λ�׶β����ͷ��ڴ��Ը���mutator�ķ��䡣��ʱ�򵥶����У���ʱҲ�����һ�α�ǽ׶β������С�


### 4.3 Remap Phase

### 4.3 Remap Phase(��ӳ��׶�)

During the Remap phase, GC threads traverse the object graph executing a read barrier against every ref in the heap. If the ref refers to a protected page it is stale and needs to be forwarded, just as if a mutator trapped on the ref. Once the Remap phase completes no live heap ref can refer to pages protected by the previous Relocate phase. At this point the virtual memory for those pages is freed.

��Remap�׶Σ�GC�̱߳�������ͼ���Զ��ڴ��е�ÿ������ִ�ж����ϡ��������ָ�����ܱ�����ҳ�棬��ô����ָ���ǳ¾ɵ�,��Ҫת����������ҵ���̵߳�������������ϵ����塣Remap�׶���ɺ󣬲����д��Ķ��ڴ�����ָ������ǰ�ض�λ�׶α�����ҳ�档��ʱ����Щҳ��������ڴ�Ҳ���ͷ��ˡ�

![](01_complete_gc_cycle.jpg)

> Figure 1: The Complete GC Cycle

> ͼƬ1: ������GC����

Since both the Remap and Mark phases need to touch all live objects, we fold them together. The Remap phase for the current GC cycle is run concurrently with the Mark phase for the next GC cycle, as shown in Figure 1.

����Remap�ͱ�ǽ׶ζ���Ҫ�Ӵ����д��Ķ�����˽����Ƿ���һ�𡣵�ǰGC���ڵ�Remap�׶�, ����һ��GC���ڵı�ǽ׶β������У���ͼ1��ʾ��

The Remap phase is also running concurrently with the 2nd half of the Relocate phase. The Relocate phase is creating new stale pointers that can only be fixed by a complete run of the Remap phase, so stale pointers created during the 2nd half of this Relocate phase are only cleaned out at the end of the next Remap phase. The next few sections will discuss each phase in more depth.

Remap�׶�Ҳ���ض�λ�׶εĺ�벿�ֲ������С��ض�λ�׶λᴴ���µĹ�ʱָ�룬ֻ�����������е�Remap�׶����޸���������ض�λ�׶ε��°벿�ִ����Ĺ�ʱָ�������һ��Remap�׶ν���ʱ������������ļ��ڽ����������ÿ���׶Ρ�

## 5. MARK PHASE

## 5. Mark �׶����

The Mark phase begins by initializing any internal data structures (e.g., marking worklists) and clearing this phase's mark-bits. Each object has two mark-bits, one indicating whether the ref is reachable (hence live) in this GC cycle, and one for it's state in the prior cycle. <sup>1</sup>

����, Mark �׶γ�ʼ�����е��ڲ����ݽṹ�����磬��ǹ����б���������˽׶εı��λ�� ÿ���������������λ�� һ����־λָʾ�ڴ�GCѭ���иö����Ƿ�ɴ�ɴＴ����һ����־λ�����ϴ�GCѭ���е�״̬��<sup>{ע1}</sup>

> <sup>1</sup> We use bitmaps for the marks, they're cheap to clear and scan.

> <sup>{ע1}</sup> ʹ��bitmap�����б�ǣ����������ɨ�衣

The Mark phase then marks all global refs, scans each threads' root-set, and flips the per-thread expected NMT value. The root-set generally includes all refs in CPU registers and on the threads' stacks. Running threads cooperate by marking their own root-set. Blocked (or stalled) threads get marked in parallel by Mark-phase threads. This is a Checkpoint; each thread can immediately proceed after it's root set has been marked (and expected- NMT flipped) but the Mark phase cannot proceed until all threads have crossed the Checkpoint.

Ȼ��, ��ǽ׶α�����е�ȫ�����ã�ɨ��ÿ���̵߳� root-set(GCroot-set��)������ת�����߳�Ԥ�ڵ�NMTֵ�� root-set ͨ������: CPU�Ĵ������߳�ջ�е��������á������е��߳�ͨ����������root-set��������Э������������ͣ�ͣ�״̬���߳���Mark-phase�̲߳��еر�ǡ�����һ������; ÿ��ҵ���߳��ڱ����root-set���Լ�Ԥ��NMT��ת��֮��Ϳ����������������Ǳ�ǽ׶ε�GC�̱߳��������ҵ���̴߳ﵽ����󣬲��ܼ������С�

After the root-sets are all marked we proceed with a parallel and concurrent marking phase [17]. Live refs are pulled from the worklists, their target objects marked live and their internal refs are recursively worked on. Note that the markers ignore the NMT bit, it is only used by the mutators. When an object is marked live, its size is added to the amount of live data in it's 1M page (only large objects are allowed to span a page boundary and they are handled separately, so the live data calculation is exact). This phase continues until the worklists run dry and all live objects have been marked.

��root-setȫ������Ǻ󣬼������в��еĲ�����ǽ׶�(��[17])���ӹ����б�����ȡ�������ã��������õ�Ŀ�������Ϊ���״̬���ݹ鴦������ڲ������á� ��ע�⣬����̻߳����NMTλ�� ����mutator�߳�ʹ�á� ��ĳ�����󱻱��Ϊ���ʱ�� ���Ĵ�С����ӵ�����1Mҳ��Ĵ���������ϣ�ֻ���������Խҳ��߽���ڣ����ҵ���������˴�����ݼ����Ǿ�ȷ�ģ����˽׶�һֱ������ֱ�������嵥�����꣬���д����󶼱�����Ϊֹ��

New objects created by concurrent mutators are allocated in pages which will not be relocated in this GC cycle, hence the state of their live bits is not consulted by the Relocate phase. All refs being stored into new objects (or any object for that matter) have either already been marked or are queued in the Mark phase's worklists. Hence the initial state of the live bit for new objects doesn't matter for the Mark phase.

������ҵ���̴߳������¶���ֻ��������ҳ���з��䣬��Щҳ�治���ڴ�GC�����б��ض�λ����� �ض�λ�׶β����漰����Щ�¶���Ĵ��״̬��־λ���¶��󣨻��κζ����е��������ã�Ҫô�Ѿ�����ǣ�Ҫô���ڱ�ǽ׶εĹ����б��е��Ŵ�����ˣ����ڱ�ǽ׶���˵���¶���Ĵ���־λ�ĳ�ʼֵ���޹ؽ�Ҫ�ġ�

### 5.1 The NMT Bit

### 5.1 NMT��־λ

One of the difficulties in making an incremental update marker is that mutators can ��hide�� live objects from the marking threads. A mutator can read an unmarked ref into a register, then clear it from memory. The object remains live (because its ref is in a register) but not visible to the marking threads (because they are past the mutator stack-scan step). The unmarked ref can also be stored down into an already marked region of the heap. This problem is typically solved by requiring another STW pause at the end of marking. During this second STW the marking threads revisit the root-set and modified portions of the heap and must mark any new refs discovered. Some GC algorithms have used a SATB invariant to avoid the extra STW pause. The cost of SATB is a somewhat more expensive writebarrier; the barrier needs to read and test the overwritten value.

�������±�ǵ�һ���ѵ��ǣ�����߳̿��ܿ�����ҵ���̲߳������Ĵ����� mutator���Խ�δ��ǵ����ö����Ĵ�����Ȼ�����ڴ������������á��ö�����Ȼ�Ǵ��״̬����Ϊ�����������ڼĴ����У�,������߳̾��ǿ���������Ϊ��mutatorջɨ������б������ˣ��� δ��ǵ�����Ҳ���ܱ���ŵ��ѱ�������С�ͨ�����ڱ�ǽ���ʱ��������һ��STW��ͣ����������⡣ �ڵڶ���STW�ڼ䣬 ����߳����·���GCroot-set�ϡ��Լ����ڴ��з����޸ĵĲ��֣����ұ����������·��ֵ����á�һЩGC�㷨ʹ��SATB����������������STW��ͣ�� SATB�Ŀ�����һ���������д����; ���д������Ҫ��ȡ����ⱻ���ǵ�ֵ��

Instead of a STW pause or write-barrier we use a read barrier and require the mutators do a little GC work when they load a potentially unmarked ref by taking an NMT-trap. We get the trapping behavior by relying on the read-barrier and the Not- Marked-Through bit: a bit we steal from each ref. Refs are 64- bit entities in our system representing a vast address space. The hardware implements a smaller virtual address space; the unused bits are ignored for addressing purposes. The read-barrier logic maintains the notion of a desired value for the NMT bit and will trap if it is set wrong. Correctly set NMT bits cost no more than the read-barrier cost itself. The invariant is that refs with a correct NMT have definitely been communicated to the Marking threads (even if they haven't yet been marked through). Refs with incorrect NMT bits may have been marked through, but the mutator has no way to tell. It informs the marking threads in any case.

���ǼȲ�ʹ��STW��ͣ��Ҳ��ʹ��д���ϣ�����ʹ�ö����ϣ������ڼ��ؿ���δ��ǵ�����ʱ��Ҫ��ҵ���߳�ִ��һ���GC��صĹ�����ͨ������NMT����ķ�ʽ������read-barrier��Not-Marked-Throughλ(��ÿ��ָ���ַ�н�ȡ��һ��bit)�������������Ϊ�� �����ǵ�ϵͳ��ָ��������64λ�ģ����Ա�ʾ�ĵ�ַ�ռ�ൽ�ò��ꣻӲ��ֻʹ�������к��ٵ�һ���������ַ�ռ�; ��Ѱַʱ�����δʹ�õ�bit�������ϵ��߼���ά��NMTλ������ֵ������������ô�����ᴥ�����塣��ȷ����NMTλ�Ŀ������ᳬ�������ϱ������������þ�����ȷ��NMT��־λ���϶��Ѵ�������̣߳���ʹ������δ����ǣ��� NMTλ��������ÿ��ܱ���ǹ�������mutator�̲߳�֪�����������κ�����¶���֪ͨ����̡߳�

If a mutator thread loads and read-barriers a ref with the NMT bit set wrong, it has found a potentially unvisited ref. The mutator jumps to the NMT-trap handler. In the NMT-trap handler the loaded value has it's NMT bit set correctly. The ref is recorded with the Mark phase logic. <sup>2</sup>  Then the corrected ref is stored back into memory. Since the ref is changed in memory, that particular ref will not cause a trap in the future. 

���һ��mutator�̼߳���һ��NMTλ��������ã��ж����ϴ��ڵĻ����ͻ��ҵ�һ�����ɷ��ʵ����á� mutator��ת��NMT���崦�������NMT���崦������У����ص�ֵ��ȷ������NMTλ���������ڱ�ǽ׶��߼��м�¼�ġ� <sup>{ע2}</sup> Ȼ�󽫸��������ô���ڴ档�������ڴ������÷����˱���� ���֮��������þͲ����ٴ������塣

> <sup>2</sup> Actually, they are batched for efficiency.

> <sup>{ע2}</sup> ʵ���ϣ�Ϊ�����Ч��,��Щ��������������ġ�

This ��self-healing�� idea is key: without it a phase-change would cause all the mutators to take continuous NMT traps until the Marker threads can get around to flipping the NMT bits in the mutators' working sets. Instead, each mutator flips its own working set as it runs. After a short period of high-intensity trapping (a ��trap storm��) the working set is converted and the mutator proceeds at its normal pace. During the steady-state portion of the Mark phase, mutators take only rare traps as their working set slowly migrates.

�ؼ����������֡������޸����Ĺ��룺 ���û�����޸����ڸ����׶��з����ĸı䣬�ᵼ������ҵ���߳���������NMT���壬ֱ������̷߳�תҵ���̹߳������е�NMTλΪֹ����ʹ�����޸�֮��ÿ��mutator������ʱ���ᷭת�Լ��Ĺ�������������ʱ��ĸ�ǿ�Ȳ�׽��������籩����֮�󣬹�������ת����ҵ���߳����������ٶ�ִ�С��ڱ�ǽ׶ε���̬�ڼ䣬mutatorsֻ���������������壬��Ϊ���ǵĹ�����Ǩ�ƻ�����

Changing the ref in memory amounts to a store, even if the stored value is Java-language-equivalent to the original value. The store is transparent to the Java semantics of the running thread, but the store is visible to other threads: without some care it might stomp over another thread's store effectively reversing it. Instead of unconditionally storing, the trap handler uses a compare-and-swap (CAS) instruction to only update the memory if it hasn't changed since the trap. If the CAS fails the handler returns the value currently in memory (not the value originally loaded) and the read barrier is repeated.

�����ڴ��е������൱�ڱ���(store)��������ʹҪ�����ֵ��Java���Բ������ԭʼֵ�� ��Java�������, ���store�������������е��߳���˵��͸���ģ����������߳��ǿɼ��ģ� ����Ҫ������һ���̻߳᲻��ȥ��ת���� ���崦�����ʹ�õ���CASָ�������������ֱ��д�룬ֻ��û�б����崦�������ĵ�����²�ȥ�����ڴ档���CASʧ�ܣ�������򷵻��ڴ��еĵ�ǰֵ�����Ѿ�����������ص��Ǹ�ֵ�ˣ������ظ������ϡ�


### 5.2 The NMT Bit and The Initial Stack-Scan

### 5.2 NMT��־λ���ʼջɨ��(Initial Stack-Scan)

Refs in mutators' root-set have already passed any chance for running a read-barrier. Hence the initial root-set stack-scan also flips the NMT bits in the root-set. Since the flipping is done with a Checkpoint instead of a STW pause, for a brief time different threads will have different settings for the NMT desired value. It is possible for two threads to throb, to constantly compete over a single ref's desired value NMT value via trapping and updating in memory. This situation can only last a short period of time, until the unflipped thread passes the next GC safepoint where it will trap, flip its stack, and cross the Checkpoint.

ҵ���߳� root-set �е������Ѿ��Ź��κ�ִ�ж����ϵĻ��ᡣ ��ˣ���ʼ root-set ջɨ��Ҳ�ᷭת�������õ�NMTλ�����ڷ�ת��ͨ�����������STW��ͣ��ɵģ�����ڶ�ʱ���ڣ���ͬ���̶߳�NMT����ֵ������ͬ�� �����߳̿��ܻ᲻�ϵ�ͨ�����������µ������õ�NMTֵ�� �������ֻ������̵ܶ�ʱ�䣬ֻҪδ��ת���߳�Ҳͨ����һ��GC��ȫ�㣬�ͻᴥ�����壬��ת��ջ�ڵ����ã���Խ�����㡣

Note that it is not possible for a single thread to hold the same ref twice in its root-set with different NMT settings. Hence we do not suffer from the pointer-equality problem; if two refs compare as bitwise not-equal, then they are truly unequal.

��ע�⣬�����߳� root-set �в����ܴ����������õĵ�ַ��ȣ�NMTֵȴ��ͬ������� ��ˣ����ǲ�������ָ���������(pointer-equality problem); ����������ð�λ�Ƚϲ���ȣ���ô���ǾͲ��ǵ�ͬ�ġ�

### 5.3 Finishing Marking

### 5.3 Finishing Marking(��ɱ��)

When the marking threads run out of work, Marking is nearly done. The marking threads need to close the narrow race where a mutator may have loaded an unmarked ref (hence has the wrong NMT bit) but not yet executed the read-barrier. Read-barriers never span a GC safepoint, so it suffices to require the mutators cross a GC safepoint without trapping. The Marking pass requests a Checkpoint, but requires no other mutator work. Any refs discovered before the Checkpoint ends will be concurrently marked as normal. When all mutators complete the Checkpoint with none of them reporting any new refs, the Mark phase is complete. If new refs are reported the Marker threads will exhaust them and the Checkpoint will repeat. Since no refs can be created with the ��wrong�� NMT-bit value the process will eventually complete.

������߳�ֹͣ����ʱ����ǽ׶μ�����ɡ� ����߳���Ҫ���ٴ���һЩ���飬����ҵ���߳̿��ܼ�����δ��ǵ����ã����NMTλ�Ǵ���ģ�����δִ�ж����ϡ� ���ϰ���Զ�����ԽGC��ȫ�㣬���ֻ��Ҫҵ���߳�ͨ��GC��ȫ��������ݽ�����͹��ˡ� ��Ǵ�����Ҫһ�����㣬������Ҫ����ҵ���̲߳��봦�� ��Checkpoint����֮ǰ���ֵ��κ����ö����������Ϊ������ ������mutators�����Checkpoint��ȴû�б����κ��µ����ã���ô��ǽ׶ξ�����ˡ� ����������µ����ã���ôMarker�߳̽��䴦��������ظ�ִ��Checkpoint�� ��Ϊ���������ò����С�����ġ�NMTλֵ�����Ա�ǹ������վ�����ˡ�


## 6. RELOCATE AND REMAP PHASES

## 6. �ض�λ�׶� �� Remap �׶����

The Relocate phase is where objects get relocated and compacted, and unused pages get freed. Recall that the Mark phase computed the amount of live data per 1M page. A page with zero live data can obviously be reclaimed. A page with only a little live data can be made wholly unused by relocating the live objects out to other pages.

�ض�λ�׶ν��ж����ض�λ���ڴ��������ͷŲ�ʹ�õ�ҳ�档 ����һ�£���ǽ׶μ���ÿ��1Mҳ���еĴ���������������Դ������Ϊ0��ҳ���ǿ��Ա����յġ� ��������������ݵ�ҳ�棬����ͨ�����������ض�λ������ҳ�棬ʹ�ô�ҳ���������ٱ�ʹ�á�

As hinted at in Figure 1, a Relocate phase is constantly running, continuously freeing memory at a pace to stay ahead of the mutators. Relocation uses the current GC-cycle's mark bits. A cycle's Relocate phase will overlap with the next cycle's mark phase. When the next cycle's Mark phase starts it uses a new set of marking bits, leaving the current cycle's mark bits untouched. The Relocate phase starts by finding unused or mostly unused pages. In theory full or mostly full pages can be relocated as well but there's little to be gained. Figure 2 shows a series of 1M heap pages; live object space is shown textured. There is a ref coming from a fully live page into a nearly empty page. We want to relocate the few remaining objects in the ��Mostly Dead�� page and compact them into a ��New, Free�� page, then reclaim the ��Mostly Dead�� page.

��ͼ1�п��Կ������ض�λ�׶β������У������ͷ��ڴ棬�Ա���������ҵ���̡߳��ض�λʱ��ʹ�õ�ǰGC���ڵı��λ��
ÿ��GCѭ�����ض�λ�׶κ���һ���ڵı�ǽ׶��в����ص�������һ���ڵı�ǽ׶ο�ʼʱ����ʹ��һ���µı��λ������ǰһ���ڵı��λ���䡣
�ض�λ�׶������Ų�δʹ�û��ߴ󲿷ֿռ�δʹ�õ�ҳ�档�����ϣ��������߻���������ҳ��Ҳ�ܽ����ض�λ����û�������塣
ͼ2��ʾ��һЩ1M���ڴ�ҳ��; ������ռ�õĲ���ʹ�����Ʊ�ʾ����ͼ2�У�ȫ�Ǵ������ҳ�����и����ã�ָ����һ�������ǿյ�ҳ�档������Ҫ����Mostly Dead��ҳ���е������������ض�λ��������������New��Free��ҳ�棬Ȼ����ա�Mostly Dead��ҳ�档

Next the Relocate phase builds side arrays to hold forwarding pointers. The forwarding pointers cannot be kept in the old copy of the objects because we will reclaim the physical storage immediately after copying and long before all refs are remapped.

���������ض�λ�׶ι��� side arrays ������ת��ָ�롣ת��ָ�벻�ܱ����ڶ���ľɸ����У���Ϊ�ڸ��ƺ���������������ڴ棬������������������ӳ��֮ǰ���кܳ�һ��ʱ�䡣

![](02_Finding_sparsely_populated_pages.jpg)

> Figure 2: Finding sparsely populated pages

![](03_Side_Arrays_and_TLB_Protection.jpg) 

> Figure 3: Side Arrays and TLB Protection

![](04_Copying_live_data_out.jpg) 

> Figure 4: Copying live data out

![](05_Updating_stale_refs.jpg)

> Figure 5: Updating stale refs


The side array data isn't large because we relocate sparse pages. We implement it as a straightforward hash table. Figure 3 shows the side array.

side array ����������������Ϊ�ض�λ����ϡ��ҳ�档����ʹ��һ���򵥵Ĺ�ϣ����ʵ������ӳ�䡣side array ��ͼ3��ʾ��


The Relocate phase then GC-protects the ��Mostly Dead�� page, shown in gray, from the mutators. Objects in this page are now considered stale; no more modifications of these objects are allowed. If a mutator loads a ref into the protected page, it's readbarrier will now take a GC-trap.

Ȼ�� �ض�λ�׶� ���� ��Mostly Dead��ҳ�棬��ֹҵ���߳��޸�,ͼ���Ի�ɫ��ʾ�� ���ڽ���ҳ���еĶ������ǳ¾ɵ�; ���������Щ��������޸ġ� ���mutator�����ü��ص��ܱ�����ҳ�棬���Ӧ�Ķ����Ͼͻ����뵽GC�����С�


Next the live objects are copied out and the forwarding table is modified to reflect the objects' new locations as shown in Figure 4. Copying is done concurrently with the mutators; the readbarrier keeps the mutators from seeing a stale object before it has finished moving. Live objects are found using the most recent mark-bits available and sweeping the page.

���������Ѵ����󿽱���ȥ, ���޸�ת�����Է�ӳ�������λ�ã���ͼ4��ʾ�� ����������ҵ���̲߳�������; �ƶ����֮ǰ��������ʹ��ҵ���̲߳��ῴ���¾ɵĶ���ʹ�����µı��λ��ɨ��ҳ������ҵ�������

Once copying has completed, the **physical** memory behind the page is freed. Virtual memory cannot be reclaimed until there are no more stale refs pointing into the freed page. Stale refs are left in the heap to be lazily discovered by running mutators using the read-barrier, and will be completely updated in the next Remap phase. Freed physical memory is immediately recycled by the OS and may be handed out to this or another process. After freeing memory, the GC threads are idled until the next need to relocate and free memory, or until the next Mark and Remap phase begins.

������ɺ󣬵ײ�� **�����ڴ�** �����ͷš� ֱ��û�й�ʱ����ָ�����ͷŵ�ҳ�棬�����ڴ��ַ�Żᱻ���ա� ���ڴ��еĹ�ʱ���ã�ʹ�������ֲ��ԣ���������ҵ���̵߳Ķ����ϣ�������һ��Remap�׶λ���ȫ���¡� �ͷŵ������ڴ��������������ϵͳ���գ��������ĳ�����̡��ͷ��ڴ��GC�߳̽����ڿ���״̬��ֱ����һ����Ҫ�ض�λ���ͷ��ڴ棬����ֱ����һ�� Mark �� Remap �׶ο�ʼ��

### 6.1 Read-Barrier Trap Handling

### 6.1 ���������崦��

If a mutator's read-barrier GC-traps, then the mutator has loaded a stale ref. The GC-trap handler looks up the forwarding pointer from the side arrays and places the correct value both in the register and in memory, as shown in Figure 5. Similarly to the NMT trap handler's ��self-healing�� behavior, updating the ref in memory is crucial to performance: it keeps the same stale ref from trapping again. As before, the memory update is done with a CAS to avoid stomping a racing store from another thread.

���ĳ���̵߳Ķ����Ͻ���GC���壬��ô������Ϊ����̼߳��ص���һ����ʱ���á� GC-trap��������side arrays�в���ת��ָ�룬������ȷ��ֵ�ŵ��Ĵ������ڴ��У���ͼ5��ʾ����NMT���崦�����ġ������޸�����Ϊ���ƣ������ڴ��е����ö�����������Ҫ��������ֹ��ͬ�Ĺ�ʱ�����ٴ��������塣��ǰ��һ����ʹ��CAS������ڴ���£��Ա�����һ���߳�������store������

It is also possible that the needed object has not yet been copied. In this case the mutator will do the copy on behalf of the GC thread �C since the mutator is otherwise blocked from forward progress. The mutator can read the GC-protected page because the trap handler runs in the elevated GC-protection mode. If the mutator must copy a large object, it may be stalled for a long time. This normally isn't an issue: pages with a lot of live data are not relocated and a `1/2`-page sized object (512K) can be copied in about 1ms.

���п�������Ķ�����δ������ɡ�����������£�ҵ���߳̽�����GC�߳�ִ�и��� - ����ҵ���߳��Ѿ����������ǰɡ� mutator��ʱ����Զ�ȡ��GC������ҳ�棬��Ϊ���崦�����������GC����ģʽ�����С����mutator���븴��һ������󣬿��᳤ܻʱ��ͣ�͡���ͨ������ʲô�����⣺���д���������ݵ�ҳ�治���ض�λ�����Ҹ��� `1/2` ҳ���С�Ķ���512K��ֻ��Ҫ����1ms��ʱ�䡣

### 6.2 Other Relocate Phase Actions

### 6.2 �ض�λ�׶ε���������

At the time we protected pages, running mutators might have stale refs in their root-set. These are already past their read-barrier and thus won't get directly caught. The mutators scrub any existing stale refs from their root-set with a Checkpoint. Relocation can start when the Checkpoint completes.

�ڱ���ҳ��ʱ�������е�mutators����root-set�п����й�ʱ�����á� ��Ϊroot-set�е�����ֱ�������˶����ϣ����Բ��ᱻֱ�ӻ�ȡ���� mutatorsʹ��һ��Checkpoint����root-set�еĳ¾����ò�����������ɺ󣬿��Լ����ض�λ��

The cost to modify the TLB protections (a kernel call and a system- wide TLB shoot-down) and scrubbing the mutators' stacks is the same for one page as it is for many. We batch up these operations to lower costs, and typically protect (and relocate and free) a few gigabytes at a time.

�޸�TLB�����ĳɱ���ϵͳ�ں˵��� + ϵͳ��Χ��TLB���䣩�Ͳ���mutatorջ����ͬ�ģ�һ������ҳ�������Ҳ��ͬ����Ϊ��Щ���������������Խ��ͳɱ���ÿ��Ҳ�ͱ�����+�ض�λ+�ͷţ�����GB���ڴ档

Notice that there is no ��rush�� to finish the Relocation phase; we need only relocate and free pages at a pace to keep ahead of the mutators. Also notice it is unlikely that a mutator stalls on an unmoved stale object. Relocated pages contain only a few older objects, most likely they have moved out of the mutator's working set. Virtual memory is not freed immediately, but we have lots of that. The final step of scrubbing all stale refs and reclaiming virtual memory is the job of the Remap phase.

��ע�⣬GC������Ҫ�����ڡ�����ض�λ�׶�; ֻ��Ҫ�ض�λ���ͷŵ��ٶȳ���mutators������ٶȾ����ˡ���Ҫע�⣬mutator��̫������δ�ƶ��Ĺ�ʱ������ͣ�١����ض�λ��ҳ��ֻ�������ٵĹ�ʱ���󣬴���������Ѿ��Ƴ���mutator�Ĺ������������ڴ治�������ͷţ���64λϵͳ�������ַ�ռ�ൽ�ò��ꡣ�������г¾����ò����������ڴ�ռ���Remap������ս׶ε�����

### 6.3 The Remap Phase

### 6.3 Remap�׶����

The Remap phase updates all stale refs with their proper forwarded pointers. It must visit every ref in the heap to find all the stale ones. As mentioned before it runs in lockstep with the next GC cycle's Mark phase; the one piece of visitor logic does both the stale ref check and NMT check.

Remap�׶θ������й�ʱ�����ã�ʹ����ȷ��ת��ָ�롣 ����������ڴ棬�����ҵ����й�ʱ�����á� ��ǰ�������������GCѭ���ı�ǽ׶�ͬ������; ��������һ���߼���ִ�й�ʱ���õļ���Լ�NMT��顣

At the end of the Remap phase, all pages that were protected before the start of the Remap phase have now been completely scrubbed. No more stale refs to those pages remain so those virtual memory pages can now be reclaimed. We also free the side arrays at this time, and a GC cycle is complete.

��Remap�׶ν���ʱ����Remap�׶ο�ʼǰ�ܵ�GC������ҳ�涼������ˡ� �����й�ʱ������ָ����Щҳ�棬���Կ��Ի�����Щ�����ڴ�ҳ�档 ��ʱҲ���ͷ���side arrays�����ˡ�һ��GCѭ����ɡ�

## 7. REALITY CHECK

## 7. ��ǰ��ʵ��

Our implementation is a rapidly moving work-in-progress. As of this writing it suffers from a few STW pauses not required by the Pauseless GC algorithm. Over time we hope to remove these STWs or engineer their maximum time below an OS time-slice quanta. We have proposed solutions for each one, and report pauses experienced by the current implementation on the 8- warehouse 20-minute pseudo-JBB run described in Section 8.

���ǵľ���ʵ�����ڱ����߸ġ� ��׫д����ʱ������ҪһЩSTW��ͣ��֧�֡���Ȼ Pauseless GC �㷨������û��STW�ġ� ����ʱ������ƣ�����ϣ����ȥ�����е�STW�������������ͣʱ��С�ڲ���ϵͳ����Сʱ��Ƭ�� ����Ϊÿ�ַ�ʽ���ṩ�˽���������ڵ�8���У�ͨ��20���ӵ�8��αJBB���У������Ե�ǰʵ�֣���ͳ������������ͣ�����

### 7.1 At the Mark Phase Start

### 7.1 ��ǽ׶εĿ�ͷ

At the start of the Mark phase we stop all threads to flip the desired NMT state. We could flip the NMT bits via a Checkpoint; the cost would be some amount of NMT-bit throbbing (repeated NMT traps) on shared objects until all threads flip. Also, global shared resources (e.g., the SystemDictionary, JNI handles, locked objects) are marked in this STW. Engineering these to use a Checkpoint is straightforward.

�ڱ�ǽ׶εĿ�ʼ����ֹͣ�����߳��Է�תNMT״̬λ������ͨ����������תNMTλ; �������ڹ�������ϻ���һ������NMTλ����(�ظ�����NMT����)��ֱ�������̶߳���תΪֹ�����⣬ȫ�ֹ�����Դ(���磬SystemDictionary��JNI�����������)�������STW�б�ǡ�ʹ�ü����ڹ�������ֱ���ּ򵥵ġ�

The worse pause reported was 21ms and the average was 16ms.

�ڲ����У����������ĵ�ʱ���� 21ms��ƽ��ʱ��Ϊ 16ms��

### 7.2 At the Mark Phase End

### 7.2 ��ǽ׶εĽ���

At the end of the Mark phase we stop all threads and do (in parallel but not concurrent) soft ref processing, weak ref processing, and finalization. Java's soft and weak refs present a race between the collector nullifying a ref and the mutator ��strengthening�� the ref. We could process the refs concurrently by having the collector CAS down a null only when the ref remains notmarked- through. The NMT-trap handler already has the proper CAS'ing behavior �C both the collector and the mutator race to CAS down a new value. If the mutator wins the ref is strengthened (and the collector knows it), and if the collector wins the ref is nullified (and the mutator only sees the null).

�ڱ�ǽ׶ν���ʱ����ֹͣ�����̣߳�(���е�������)ִ�������ô���(soft ref)�������ô���weak ref�����ս�(finalization)��Java�е������ú������ã���GC�ͷ����á���ҵ���߳̽�����ǿ����֮����ھ����� ���ǿ��Բ����ش������ã�ͨ�����ռ���ֻ������δ���ʱͨ��CAS��������Ϊnullֵ����Ӧ�� NMT-trap ��������Ѿ�������ȷ��CAS��Ϊ ���� �ռ�����mutator��������CAS��Ϊһ����ֵ�����mutator�ɹ���ref�ͻ���ǿ(�ռ����ܸ�֪��)������ռ����ɹ������þͻ��ÿ�(��mutator�ῴ��null)��

There are a number of other items handled at this STW that could be engineered to be concurrent, including class unloading and code-cache unloading. Again engineering these will be straightforward but tedious.

��STW�еĺܶദ�����ǿ������Ϊ�����ģ�������ж�غʹ��뻺��ж�ء�ͬ���������Щ������Ȼ�򵥵�Ҳ����ʱ�䡣

The worse pause reported was 16ms and the average was 7ms.

�ڲ����У����������ĵ�ʱ���� 16ms��ƽ��ʱ��Ϊ 7ms��

### 7.3 At the Relocation Phase Start

### 7.3 �ض�λ�׶ο�ʼʱ

The mutators' root-sets need scrubbing when GC-protecting a page. There are two problems here: the TLB shoot-down isn't atomic and there are stale refs in the root-set. Since the TLB shoot-down is not atomic, for a brief period some mutators can be protected and not others. Unprotected mutators would continue to read and write the object directly, so protected mutators need to as well. However, reading and writing the protected object forces a GC-protection trap. Our current implementation stops all threads and performs a bulk TLB shoot-down and mutator root-set scrubbing under STW. This can be engineered to be concurrent and incremental in a straightforward manner.

��GC����ĳ��ҳ��ʱ��ҵ���̵߳�root-sets��Ҫ���� ������������: TLBж�ز���ԭ�Ӳ�������root-sets�л��г¾ɵ����á�����TLBж�ز���ԭ���Եģ��ڼ��̵�ʱ���ڣ�������һ����ҵ���߳��ܵ�����������һ���ֲ��ܱ��������ܱ�����ҵ���߳̽�����ֱ�Ӷ�д��������ܱ�����ҵ���߳�Ҳ��Ҫ��������Ȼ������ȡ��д���ܱ��������ǿ�ƽ���GC�������塣���ǵ�ǰ��ʵ����ֹͣ�����̣߳�����STW��ִ������TLBɾ�����̵߳�root-set����Ҳ����ֱ�ӽ������Ϊ�����ģ�֧�������ķ�ʽ��

We could use a Checkpoint to update the TLBs and scrub the root-sets. To maintain concurrency until all threads have passed the relocation Checkpoint, the read barrier's TLB trap handler is modified to wait for the Checkpoint to complete before proceeding with relocation or remapping and propagating a corrected ref in the mutator. Mutator threads that actually access refs in protected pages will then ��bunch up�� at the Checkpoint with other threads continuing concurrent execution past the Checkpoint. This effect is mitigated by the fact that we preferentially relocate sparse pages.

���ǿ���ʹ��Checkpoint������TLB������root-sets�� Ϊ�˱��ֲ���, �������̶߳�ͨ���ض�λ����֮ǰ�������ϵ�TLB���崦�������Ҫ�ȴ�������ɣ�Ȼ������ض�λ����ӳ�䣬����mutator�д�������������á� ʵ�ʷ����ܱ���ҳ���е����õ�Mutator�߳̽��ڼ��㡰�ۼ����������߳���ͨ�������������ִ�С� ���������ض�λϡ��ҳ�棬Ҳ�ͼ���������Ӱ�졣

The worse pause reported was 19ms and the average was 5ms.

�ڲ����У����������ĵ�ʱ���� 19ms��ƽ��ʱ��Ϊ 5ms��

### 7.4 Relocate doesn't run during Mark/Remap

### 7.4 �� Mark/Remap �ڼ仹����ִ���ض�λ

Right now we have not implemented a second set of mark bits to allow the Relocate phase to run concurrently with the next Mark/Remap phase [14]. This means we cannot free memory during the Mark/Remap phase. We have heuristics which predict how many pages the mutator will need during marking and we free that many (plus some pad) before marking begins. If we predict low, as can happen if the mutators suddenly ��accelerate��, the mutators will block until marking is complete. Engineering the overlapped Relocate/Mark phases will be straightforward. Additionally, we currently do not add threads dynamically in response to mutator acceleration. Each phase completes with a number of threads decided on at the phase start.

��ֹ���ķ���ʱ����δʹ�õڶ�����λ��ʵ�֣��������ض�λ�׶�����һ��GC���ڵ�Mark/Remap�׶�(��[14])����ִ�С� Ҳ����˵�ھ���ʵ������ʱ��������Mark/Remap�׶��ͷ��ڴ档 ����ʹ������ʽ�㷨����Ԥ��ҵ���߳��ڱ���ڼ���Ҫ����ҳ�棬���ڱ�ǿ�ʼǰ�ͷ��㹻��ҳ�棨����һЩ��䣩�� ���Ԥ����ˣ�����ҵ���߳�ͻȻ�����١�����ҵ���̻߳�������ֱ�������ɡ� �� �ض�λ/Mark �׶����Ϊ����ִ�зǳ��򵥡� ���⣬ĿǰҲ������ҵ��ͻȻ����ʱ��̬���GC�̡߳� ÿ���׶ε��߳����ڸý׶ο�ʼʱ��ȷ���ˡ�

## 8. EXPERIMENTS

## 8. ʵ����֤

### 8.1 Methodology

### 8.1 ������

The Pauseless algorithm is intended to lower pause times in large transaction-oriented programs running business logic. There are a limited number of representative Java benchmarks for this class of program. The most realistic and widely accepted is SpecJApp-Server '02 and '04. This benchmark is extremely difficult to setup, tune, or get reliable numbers out of. It is also very hard to normalize across different hardware. The much more simplistic SpecJBB benchmark has very well-structured (and unrealistic!) object lifetimes and is ideally suited for a generational collector.

Pauseless�㷨ּ�ڽ�����������Ĵ���ҵ��ϵͳ����ͣʱ�䡣��һ��ϵͳ����ʹ�õ�Java��׼���ߺ��١�����ʵ��ʹ����㷺���� SpecJApp-Server '02 �� '04�� �������׼���Էǳ��������ã��������ÿɿ������֡� ��ƽ̨Ҳ�ǳ����ѡ����и����򵥵� SpecJBB ��׼���ԣ����нṹ���õĶ����������ڣ�����������ʵ��Ӧ�ó��������ǳ��ʺϷִ������ռ�����

In an effort to have both a reliable, understandable benchmark and one that is more representative of transactional programs, we added a large object cache to the standard SpecJBB benchmark. This cache represents, e.g., a Java bean cache, or an HTML request cache. For each transaction, 400 bytes were added to the cache and the oldest cached object was freed. This level of extra objects is enough to easily defeat targeted tuning of generational collectors to JBB.

Ϊ�˻�ÿɿ��������Ļ�׼���ԣ���Ҫ�ܴ���ʵ�ʵ�ҵ������� �����ڱ�׼��SpecJBB��׼�����������һ������󻺴档�û�����Ա�ʾ��Java bean ���桢����HTML���󻺴档����ÿ�����񣬽�400�ֽ���ӵ������У����ͷ����ϵĻ������ ���ֶ���Ķ����������׵���ֹ���JBB�ķִ������ռ�����������Եĵ�����

We also removed the forced System.gc() between runs and increased the JBB run times from 2 minutes to 20 minutes. <sup>3</sup>
In the standard benchmark it's common to never need a full collection during the timed portion of the run. In practice, these large business applications must run in a steady-state mode without an untimed window every 2 minutes for a System.gc().

���ǻ��Ƴ��˸���run֮��� `System.gc()`������JBB������ʱ���2�������ӵ�20���ӡ�<sup>3</sup>
�ڱ�׼��׼�����У�ͨ�������е��ض��ڼ䲢����Ҫ�����������ռ����������������У���Щ����ҵ��ϵͳ����ƽ�����У�������ÿ��2���Ӿ���һ��GC���ڡ�

> <sup>3</sup> Except for IBM's concurrent collector which was unable to run the full 20 minutes; we used a 10 minute run for it.

> <sup>3</sup> ��IBM�Ĳ����ռ������⣬���޷�����������20����; ����ֻ�ܽ�������ʱ������Ϊ10���ӡ�


All runs were done with 8 warehouses, i.e. 8 concurrent threads doing benchmark work. We added ��-Xmx1536m��, allowing a maximum heap size of 1.5G, which is about twice the average size of the live data. We added ��-server�� to the SUN JVMs. For the concurrent GC timing runs, we added whatever flag was appropriate to trigger using the concurrent collector for that JVM. For the IBM JVM, it was ��-Xgcpolicy:optavgpause��. For the BEA JVM, it was ��-Xgcprio:pausetime��. For the SUN JVM, it was ��-XX:+UseConcMarkSweepGC -XX:+UseParNewGC��. For the Azul JVM, concurrent collection is the default and no flags are needed. For the non-concurrent GC timing runs we used the best parallel (throughput-oriented) collector available. This is the default for the IBM and BEA JVMs, for the SUN JVM we added ��-XX:+UseParallelGC��. We used no other flags.

���е�run����8���ֿ���ɣ���8�������߳̽��л�׼���ԡ���������� ��-Xmx1536m�������������ڴ�Ϊ1.5G����Լ��ƽ�������������������������SUN JVM������ˡ�-server�����������ڲ���GC��run������������ʺϸ�JVM�Ĳ����ռ���������־������IBM JVM��˵�ǡ�-Xgcpolicy:optavgpause��������BEA JVM���ǡ�-Xgcprio:pausetime��������SUN JVM���ǡ�-XX:+UseConcMarkSweepGC -XX:+UseParNewGC��������Azul JVM�������ռ���Ĭ�����ã�����Ҫָ�������ڷǲ���GC��run������ʹ�ã������������ģ���õĲ����ռ���������IBM��BEA JVM��Ĭ�����ã�����SUN JVM��������ˡ�-XX:+UseParallelGC���������������ǲ�û��ָ����

We ran the IBM and SUN JVMs on a 2-way 3.2Ghz hyperthreaded Xeon with 2G of physical memory, running a Red Hat Linux 2.6 kernel. Unfortunately, the BEA JVM didn't run on this version of Linux so it was run on a 1-way 2.4Ghz hyperthreaded P4 with 512M of physical memory running Windows 2000. The BEA JVM heap was limited to 425M to avoid paging. The simulated object cache added about 40M of long-lived live data per warehouse; 425M isn't a large enough heap to run with 8 warehouses. We limited the BEA JVM to 3 warehouses, keeping the proportion of heap devoted to long-lived data about the same. We also ran the SUN JVM in 64-bit mode on a 2-way 1.2Ghz US3 with 4G of physical memory running Solaris 9. We attempted to run on an older 24-CPU Sparc (450Mhz US2). Here we hoped the Sparc would use the spare CPUs to good effect. However, the single-threaded concurrent collector could not keep up with the mutators and the benchmark suffered numerous 12-second full-GC pauses. On the 2-CPU Sparc, a single concurrent collector thread could use up to half the total CPU resources in order to keep up. We report the superior 2-CPU Sparc scores, although we would like to have reported scores from another high-CPU count machine. The Azul JVM is a 64- bit JVM running on a 16-chip (384-CPU) Azul appliance with 128G of physical memory. As before, we limited heap size to 1.5G. Only 8 CPUs are used to run the actual benchmark, with a handful more running the Pauseless collection and doing background JIT compiles.

������2·3.2Ghz���߳�Xeon������������IBM��SUN��JVM���ڴ�Ϊ2GB, ����ϵͳ��Red Hat Linux 2.6�ںˡ����ߵ��ǣ�BEA JVM ��֧������汾��Linux�������ֻ�������ڵ�·2.4Ghz���߳�P4�������ϣ������ڴ�512M��ϵͳΪWindows 2000. BEA JVM�Ķ��ڴ�����Ϊ425M�Ա���ʹ�ý����ڴ档 Ϊÿ���ֿ�ģ��Ķ��󻺴��Լ��40M�ĳ��ٴ������; 425M�Ķ��ڴ沢����������8���ֿ⡣���ǽ�BEA JVM����Ϊ3���ֿ⣬�������ڳ������ݵĶѵı���������ͬ�����ǻ���2·1.2Ghz US3�Ļ���������64λģʽ��SUN JVM�����������ڴ�Ϊ4G��ϵͳ��Solaris 9. ���ǳ����ڽϾɵ�24-CPU Sparc��450Mhz US2��ƽ̨�����С����������ϣ��Sparc�ܹ�ʹ�ñ���CPU���ﵽ���õ�Ч�������ǣ����̲߳����ռ����޷�����mutator�����󣬲��һ�׼����ʱ�����˴�����12�� FullGC ��ͣ����2-CPU Sparc�ϣ���������GC�߳���Ҫ������CPU��Դ��һ����ܸ���ҵ���̵߳���Ҫ�����Ǳ����������2-CPU Sparc��������������ϣ��������һ̨��CPU���������ķ����� Azul JVM��һ��64λJVM, ������16���壨384-CPU����Azul�豸�ϵģ�����128G�������ڴ档����ǰһ�������ǽ��Ѵ�С����Ϊ1.5G��ֻ��8��CPU��������ʵ�ʻ�׼���ԣ���������Pauseless GC �����к�̨JIT���롣

We decided to NOT report SpecJBB score, which is reported in units of transactions/second, both because our run is not Speccompliant and because of the wide variation in hardware and JIT quality. Even on the same hardware, the JITs from different vendors produce code of substantially different quality. For the same 20 minute run, we saw JVMs execute between 15 million and 30 million transactions. While transaction throughput is an important metric, this paper is focused on removing the biggest reason for transaction time variability. We report transaction times instead.

���Ǿ���������SpecJBB�÷֣����ĵ�λ�� TPS��transactions/second������Ϊ���ǵ����в���Speccompliant��������ΪӲ����JIT�Ĳ���ܴ󡣼�ʹ����ͬ��Ӳ���ϣ����Բ�ͬ��Ӧ�̵�JITҲ��������������ܴ�Ĵ��롣 ����ͬ��20���������У����ǿ���JVMִ����1500��3000���ҵ����Ȼ��������һ����Ҫ��ָ�꣬�����ĵ��ص���������������֮���ʱ��仯�����Ա�����ҵ����Ӧʱ�䡣

### 8.2 Transaction Times

### 8.2 ҵ����Ӧʱ��

We measured both transaction times and GC pause times reported with ��-verbose:gc��. We feel that transaction times represent a more realistic measure than direct GC pauses as they more closely correspond to ��user wait time��.

ʹ�� ��-verbose:gc�� ѡ��������ҵ����Ӧʱ���GC��ͣʱ�䡣 ������Ϊҵ����Ӧʱ���ֱ�ӵ�GC��ͣ������ʵ���������Ϊ���ӽ��ڡ��û��ȴ�ʱ�䡱��

Transaction times were gathered into buckets by duration, building a histogram. Duration was measured with Java's current- TimeMillis() and so is limited to millisecond resolution. Most transactions take 0 or 1 milliseconds, so we did not gather accurate times for these fast transactions. However, we are more interested in the slow transactions. All the collectors except Pauseless had a significant fraction of transactions take 100- 300ms (100 times slower than the fast transactions), with spikes to 1-4 seconds. We kept per-millisecond buckets from 0ms to 31ms. After that we grew the buckets by powers-of-2 with halves: 32-47ms, 48-63ms, 64-95ms, 96-127ms, and so on up to 16sec. This allowed us to compute the bucket index with a few shifts. Buckets were replicated per thread to avoid coherency costs then totaled together at the end of the run.

ҵ����Ӧʱ�䰴����ʱ���ռ���Ͱ�У�����ֱ��ͼ��ʹ��Java�� `currentTimeMillis()` ������������ʱ�䣬��˾���Ҳ������Ϊ���뼶�������ҵ��ֻ��Ҫ0-1���룬�������û���ռ���Щ���������׼ȷʱ�䡣���ǣ����Ƕ�����Ӧ��ҵ�������Ȥ����Pauseless֮������������ռ��������൱һ���ֵ�ҵ����Ҫ�ķ�100-300ms���ȿ��ٵ�ҵ��Ҫ��100�����ϣ�����ֵΪ1-4�롣���ǽ�ÿ�����Ͱ������0ms-31ms֮�䡣����֮��������2��2����������Ͱ��32-47ms��48-63ms��64-95ms��96-127ms���������ƣ��16�롣�����������ü������������Ͱ������ÿ���̸߳��ƴ洢Ͱ�Ա���һ���Կ�����Ȼ�������н���ʱ���ܡ�

A transaction that reports as taking 0ms clearly takes some finite time. The 0ms bucket's average transaction time is assumed to be 0.33ms, and the 1ms bucket's average transaction time is assumed to be 1.33ms. This is the largest source of measurement error we have. Almost no transactions landed in the 3ms to 30ms buckets, so a measurement error of up to 1ms in those buckets will not alter the data in any substantial way.

����Ϊ0ms��ҵ����ȻҲ����Ҫһ������ʱ��ġ�����0msͰ��ƽ������ʱ��Ϊ0.33ms������1msͰ��ƽ������ʱ��Ϊ1.33ms���������ǲ������������Դ������û���κ���������3ms-30ms��Ͱ�У������ЩͰ�еĲ�������ߴ�1msҲ����ʵ���Եظı����ݡ�

For all other buckets we simply totaled time for that bucket. We summed the total transaction times (time per bucket by transactions in the bucket), and report the percentage of total transaction time spent on transactions of each duration.

��������Ͱ�����Ǽ򵥵ػ�����ÿ��Ͱ��ʱ���ܺ͡���Ͱ��ÿ��������ܺͣ�����������ÿ��ʱ��ε����������ѵ�����Ӧʱ��İٷֱȡ�

Figure 6 shows how many transactions the various JVMs kept in the 0ms and 1ms range (0ms is the low bar, 1ms is the middle bar). The Pauseless algorithm keeps 87% (99.5%) of total transaction time spent in transactions of 1ms (2ms) or less; the other JVMs vary between 80% down to 50%. The concurrent version from each vendor faired slightly worse than the parallel collectors, showing a slightly higher percentage of total time spent in slow transactions.

ͼ6��ʾ����0ms��1ms��Χ�ڸ���JVM�ܳ�������������0ms���·�������1ms���м�������� Pauseless�㷨��1ms��2ms������̵�ʱ�����������ı����ߴ�87����99.5����;����JVM��80����50��֮��仯��ÿ�����̵Ĳ����汾���Ȳ��а汾�Բ�һЩ����ʾ��ͼ�о�������������ռ�ı����Ըߡ�

Figure 7 shows cumulative transaction times (not wall-clock time, which was 20 minutes) vs. transaction duration. Times are cumulative, reaching 1.00 (100% of total transaction time) at the top edge. Transaction duration runs across the bottom in a log scale. Lines that approach 1.00 quicker are better, representing a greater percentage of processing time spent in fast transactions.

ͼ7��ʾ���ۼƵ�ҵ����ʱ�䣨���ǹ���ʱ�䣬��20���ӣ��뽻�׳���ʱ��Ĺ�ϵ��ʱ�����ۻ��ģ�����ߴﵽ1.00����ҵ����Ӧʱ���100���������׳���ʱ���Զ���������Խ�ײ���Խ��ӽ�1.00��Խ�ã���ʾ�ڿ��ٽ����л��ѵ�ʱ���������

We can see a couple of trends in this chart. Pauseless again does quite well, with essential 100% of time spent in fast transactions and a worst-case transaction time of 26 milliseconds. The other JVMs are roughly grouped into pairs with the parallel throughput collector line being slightly higher than the concurrent collector line for most of the chart. The lines cross as we near 100% of time and the slowest transactions; the concurrent collectors generally have smaller worst-cast times than the throughput collectors.

���ǿ����ڴ�ͼ���п����������ơ� Pauseless �����൱��������100����ʱ�仨�ڿ��ٽ����ϣ������µ�ҵ����Ӧʱ��Ϊ26���롣����JVM���³ɶԷ��飬�����ͼ���У������ռ������������Ը��ڲ����ռ��������ӽ�100����ʱ��������Ľ���ʱ����Щ�߷�������; �����ռ�����������ͨ�����������ռ�������ͣʱ��ҪСһЩ��

Table 1 shows the worse-case transaction times. The Pauseless algorithm's worse-case transaction time of 26ms is over 45 times better than the next JVM, BEA's parallel collector. Average transaction times are remarkable similar given the wide variation in hardware used.

��1��ʾ�������µ�ҵ����Ӧʱ�䡣 Pauseless�㷨��������������26ms������һ��JVM��BEA�Ĳ����ռ���������45�������ǵ�����Ӳ������һ����ƽ��ҵ����Ӧʱ����ܴ�����ʵ�����

![](06_01_Worst-case_and_average_times.jpg)

> Table 1: Worst-case and average times, in ms

![](06_02_Short_transaction_times.jpg)

> Figure 6: Short transaction times (0,1,2 ms) as a % of total

![](07_01_Cumulative_transaction_times_vs_duration.jpg)

![](07_02_Cumulative_transaction_times_vs_duration.jpg)

![](07_03_Cumulative_transaction_times_vs_duration.jpg)

![](07_04_Cumulative_transaction_times_vs_duration.jpg)

> Figure 7: Cumulative transaction times vs. duration (ms)


![](08_01_Reported_pause_times_vs_duration.jpg)

![](08_02_Reported_pause_times_vs_duration.jpg)

![](08_03_Reported_pause_times_vs_duration.jpg)

![](08_04_Reported_pause_times_vs_duration.jpg)

> Figure 8: Reported pause times vs. duration (ms)



### 8.3 Reported Pause Times

### 8.3 ��ͣʱ�䱨��

We collected GC pause times reported with ��-verbose:gc��. We summed all reported times and present a histogram of cumulative pause times vs. pause duration. Figure 8 shows the reported pauses. Most of the concurrent collectors consistently report pause times in the 40-50ms range; IBM's concurrent collector has 150ms as it's common (mode) pause time. As expected, the parallel collectors all do worse with the bulk of time spent in pauses ranging from 150ms to several seconds.

����ʹ�á�-verbose��gc�����ռ�GC��ͣʱ��ı��档 ��������ͣʱ����ӣ�����Ϊһ�� �ۻ���ͣʱ��VS.��ͣ����ʱ���ֱ��ͼ�� ͼ8��ʾ����ͣ���档 ����������ռ�������ͣʱ��һֱ��40-50ms��Χ��; IBM�Ĳ����ռ�����Ϊ������ģʽ����ͣʱ����ﵽ150�������ҡ� ����Ԥ�ڵ������������ռ�����ͣ��ʱ�䷽����ֺ���⣬�Ӵ���150msֱ������֮�䶼�С�

Table 1 also shows the ratio of worst-case transaction time and worst-case reported pause times. Note that JBB transactions are highly regular, doing a fixed amount of work per transaction. Changes in transaction time can be directly attributed to GC.<sup>4</sup> 
Several of the worse-case transactions are a full second longer than the worse-case pauses. We have some guesses as to why this is so:

��1����ʾ����������ʱ����������ͣʱ��ı��ʡ� ��ע�⣬JBB�����Ƿǳ�����ģ�ÿ������ִ�й̶������Ĺ����� ����ʱ��ı仯����ֱ�ӹ�����GC��<sup>4</sup>
ĳЩ�����Ľ��ױ���������ͣʱ�䳤һ���롣 ������һЩ�²⣬Ϊʲô��������

> <sup>4</sup> We tested; all transactions are fast until the heap runs out. For the 64-bit JVMs we were able to test with a 64G heap.

> <sup>4</sup> ���ǲ�����; �ڶѺľ�֮ǰ���������񶼺ܿ졣 64λJVM֧��ʹ��64G�Ķ��ڴ������в��ԡ�

It is possible that the concurrent collectors did not keep up with the allocation rate, stalling mutators until they caught up. Unfortunately, this information was not obvious from the ��-verbose: gc�� output. Also, during some phases of some concurrent GCs, the mutators pay a heavy cost while making forward progress. This amounts to an unreported pause smeared out in time. Sometimes the GC pauses come in rapid succession so that the same transaction will get paused several times. Perhaps the underlying OS timesliced the 8 mutator threads very poorly across the 4 hyper-threaded CPUs.

�п����ǲ����ռ������ٶȸ����Ϸ����ٶȣ�������ҵ���̡߳����ҵ��ǣ������Ϣ�ڡ�-verbose: gc��������в������ԡ����⣬��ĳЩ����GC�Ĳ��ֽ׶��У�ҵ���߳��ڲ���ִ��ʱ�Ḷ�����صĴ��ۡ����൱����һ����δ�������ͣʱ�䱻ͿĨ�ˡ���ʱ����������ش���GC��ͣ�����ͬһ��������ܻᱻ��ͣ�ü��Ρ����п����ǵײ�ֻ��4�����߳�CPU�ںˣ�����ϵͳ��8��ҵ���̷߳����˷ǳ��̵�ʱ��Ƭ��

In any case, **reported pause times can be highly misleading**. 
The concurrent collectors other than Pauseless under-report their effects by 2x to 6x! The parallel collectors also under- report, but only by 30% to 100%. Based on this data, we encourage the GC research community to test the end-to-end effects of GC algorithms carefully.

������Σ�**��ͣʱ�䱨����ܻ�����ܴ����**��
����Pauseless֮������������ռ������ܻ��ٱ���2x��6x����Ӱ�죡�����ռ���Ҳ���ٱ��棬��ֻ��30����100����������Щ���ݣ����ǹ���GC�о��Ŷ���ϸ����GC�㷨�Ķ˵���Ч����

We also attempted to gather Minimum Mutator Utilization figures [12], especially to track the ��trap storm�� effects. MMU reports the smallest amount of time available to the mutators in a continuous rolling interval. Since our largest pause was over 20ms there exists a 20ms interval where the mutators make no progress, so MMU@20ms is 0. Preliminary figures are in Table 2, and represent MMU figures for the entire 20 minute run worst case across all threads. Looking at the MMU@50ms figure, we see about 40ms of pause out of 50ms. We know that about 20ms of that is reported as an STW pause, so we assume the remaining 20ms is due to the trap storm.

���ǻ������ռ� Minimum Mutator Utilization ���ݣ���[12]�����ر���׷�١�����籩��ЧӦ�� MMU�������������������ҵ���߳̿��õ���Сʱ�������������ǵ����ͣ�ٳ���20ms����˴���20ms�ļ��������ҵ���߳�û�н�չ����� MMU@20ms ֵΪ0. ���������ڱ�2�У�����MMU��ֵ���������߳�������20���������е������� �ٿ� MMU@50ms ��ֵ�����Է�����50ms�ڴ�Լ��40ms����ͣ������֪�����д�Լ 20ms ������ΪSTW��ͣ���������Ǽٶ�ʣ�µ�20ms��������籩����ġ�

![](08_05_Minimum_Mutator_Utilization.jpg)

> Table 2: Minimum Mutator Utilization

## 9. Conclusions

## 9. �ܽ�

Azul Systems has taken the rare opportunity to produce custom hardware for running a garbage collected language in a shipping product. This custom hardware enables a very potent garbage collection algorithm. Even though the individual Azul CPUs are slower than the high-clocking X86 P4's compared against, worse-case transaction times are over 45 times better and average transaction times are comparable.

Azul Systems �����ѵõĻ���, ���ƻ�������Ӳ���豸�����������ռ�����, ������װ�˲�Ʒ��(shipping product)�����ֶ��Ƶ�Ӳ��ʵ���˷ǳ���Ч�������ռ��㷨�����ܵ����� Azul CPU ��Ƶ�ȸ�Ƶ�� X86 P4 ��һЩ���������µ�ҵ����ʱ��ȴ�� P4 ����45������ƽ��ҵ����ʱ���������ࡣ

Azul's Pauseless GC algorithm is a fully parallel and concurrent algorithm engineered for large multi-processor systems. It does not need any Stop-The-World pauses, no places where all mutator threads must be simultaneously stopped. Dead object space can be reclaimed at any point during a GC cycle; there are no phases where the GC algorithm has to ��race�� to finish some phase before the mutators run out of free space. Also there are no phases where the mutators pay a continuous high cost while running. There are brief ��trap storms�� at some phase shifts, but due to the ��self-healing�� property of the algorithm these storms appear to be low cost.

Azul��Pauseless GC�㷨����Ϊ���Ͷദ����ϵͳ��Ƶģ�֧�ֲ�������ȫ���е�GC�㷨��������Ҫ�κε�Stop-The-World��ͣ��û��ʲô�ط���Ҫֹͣ���е� mutator�̡߳���GCѭ���ڼ������ʱ�̣������Ի�����������ռ�õĿռ�; ��ҵ���̺߳ľ����п��ÿռ�֮ǰ, Ҳû���ĸ��׶���GC�㷨���롰���š�ȥ��ɵġ���ȻҲû���ĸ��׶���Ҫҵ���߳����������߰��ĳɱ�����ĳЩ�׶α任ʱ����ڶ��ݵġ�����籩�����������㷨�ġ������޸������ԣ���Щ�籩�Ĵ���Ҳ�ǵͳɱ��ġ�

Azul's custom hardware includes a read-barrier, an instruction executed against every ref loaded from the heap. The read-barrier allows global GC invariants to be cheaply maintained. It checks for loading of potentially unmarked objects, preventing the spread of unmarked objects into previously marked regions of the heap. This allows the concurrent incremental update Mark phase to terminate cleanly without needing a final STW pause. The read-barrier also checks for loading stale refs to relocated objects and it does it cheaper than a Brooks' style indirection barrier.

Azul ���Ƶ�Ӳ������һ�������ϣ�һ����ԴӶ��ڴ��������ʱִ�е�ָ���������������ά��ȫ�ֵ�GC������������������δ��Ƕ���ļ��أ���ֹδ��Ƕ�����ɢ����ǰ��ǵ������С����������������±�ǽ׶θɴ��ֹͣ��������Ҫ�������һ��STW��ͣ�������ϻ�����¾ɵ�ָ�룬���ض�λ�Ķ��󣬱��� Brooks ���ļ�����ϴ��۸��͡�

Section 7, Reality Check, includes ongoing and future work. Another obvious and desirable feature is a generational variation of Pauseless. As presented, Pauseless is a single-generation algorithm. The entire heap is scanned in each Mark/Remap cycle. Because the algorithm is parallel and concurrent, and we have plentiful CPUs the cost is fairly well hidden. On a fully loaded system the GC threads will steal cycles from mutator threads, so we'd like the GC to be as efficient as possible. A generational version will only need to scan the young generation most of the time. The necessary hardware barriers already exists.

�ڡ���7�� ��ʵ״�����У����������ڽ��еĺ���Ҫ����Ĺ�������һ�����������㽵������Ƿִ�����Pauseless������ǰ������֪�� Pauseless ��һ�ֲ��ִ���GC�㷨��ÿ�� Mark/Remap ѭ������ɨ�������ѡ���Ϊ�㷨�ǲ��кͲ����ģ���������ӵ�г����CPU��Դ�����Կ����൱�����ԡ����������ص�ϵͳ�ϣ�GC�̻߳���mutator�߳�����CPUʱ�ӣ��������ϣ��GC�����ܵظ�Ч������������ִ��汾����󲿷�ʱ��ֻ��Ҫɨ������������ˡ�����ص�Ӳ������Ҳ�Ѿ����ˡ�

On a final note, we were quite surprised at the difference between reported pause times and the ��user experience�� delays seen by the transactions. We strongly encourage GC researchers and the production JVM providers to pay close attention to full GC algorithm costs, not just those costs that can easily have a timerstart/ timer-stop wrapped around them.

������ǶԱ������ͣʱ��, ��ʵ��ҵ��ġ��û����顱�ӳ�֮��Ĳ���е��ǳ����ȡ�ǿ�Һ���GC�о���Ա�Ͳ�Ʒ��JVM�ṩ�̽��ܹ�ע������GC�㷨�ɱ��������������� timerstart/timer-stop �Ϳ����������������Ĳ��֡�


## 10. REFERENCES

## 10. �ο�����

- [1] Agesen, O. GC Points in a Threaded Environment. SMLI TR-98-70. Sun Microsystems, Palo Alto, CA. December 1998.

- [2] Appel, A., Ellis, J., Li, K., Real-time concurrent collection on stock multiprocessors. In 1988 Conference on Programming Language Design and Implementation (PLDI), June 1988.

- [3] Appel, A., Li, K., Virtual Memory Primitives for User Programs. In 1991 Conference on Architectural Support for Programming Languages and Operating System, April 1991.

- [4] Bacon, D., Cheng, P., Rajan, V. The Metronome: A simpler approach to garbage collection in real-time systems. In Proceedings of the OTM Workshops: Workshop on Java Technologies for Real-Time and Embedded Systems, Catania, Sicily, Nov. 2003.

- [5] Baker, H., List processing in real time on a serial computer, Communications of the ACM, Vol. 21, 4, (April 1978),.280-294

- [6] Barth, J. Shifting garbage collection overhead to compile time. Communications of the ACM, Vol. 20, 7 (July 1977), 513- 518

- [7] BEA Systems. 2003. BEA JRockit: Java for the Enterprise. White paper. BEA Systems, San Jose, CA.

- [8] Blackburn, S., McKinley, K., In or Out? Putting Write Barriers in Their Place. In Proceedings of the 2002 International Symposium on Memory Management, Berlin, Germany, 2002.

- [9] Blackburn, S., Hosking, A., Barriers: Friend or Foe? In Proceedings of the 2004 International Symposium on Memory Management, Vancouver, Canada, 2004.

- [10] Brooks, R. Trading data space for reduced time and code space in real-time garbage collection on stock hardware. In 1984 ACM Symposium on Lisp and Functional Programming. (Aug. 1984) 256-262

- [11] Cheney, C. A Nonrecursive List Compacting Algorithm. Communications of the ACM, Vol. 13, 11 (Nov. 1970), 677-678

- [12] Cheng, P., Blelloch, G., A parallel, real-Time garbage collection. In Conference on Programming Languages Design and Implementation (PLDI '01). Snowbird, Utah, June 2001

- [13] Collins, G., A method for overlapping and erasure of lists. Communications of the ACM, Vol. 3, 12 (Nov. 1960), 655-657

- [14] Detlefs, D., Flood, C., Heller, S., Printezis, T. Garbage-first garbage collection. In Proceedings of the 2004 International Symposium on Memory Management, Vancouver, Canada, 2004

- [15] Deutcsh, P., Bobrow, D. An Efficient, Incremental, Automatic Garbage Collector. Communications of the ACM, Vol. 19, 9 (Sept. 1976), 522-527

- [16] Fenichel, R., Yochelson, J. A LISP Garbage-Collector for Virtual Memory Systems. Communications of the ACM, Vol. 12, 11 (Nov. 1969), 611-612.

- [17] Flood, C., Detlefs, D., Shavit, N., Zhang, C. Parallel Garbage Collection for Shared Memory Multiprocessors. In 2001 USENIX Java Virtual Machine Research and Technology Symposium (JVM ��01). Monterey, CA, April 2001

- [18] Goa, H., Nilsen, K. The real-time behavior of dynamic memory management in C++. In Proceedings of the Real-Time Technology and Applications Symposium. Chicago, IL, 1995

- [19] Gosling, J., Bollela, G. The Real-Time Specification for Java. Addison-Wesley, Boston MA, 2000.

- [20] Heil, T., Smith, J. Concurrent garbage collection using hardware-assisted profiling. In Proceedings of the 2nd International Symposium on Memory Management, Minneapolis, MN, 2000

- [21] Hosking, A., Moss, E., Stefanovic, D., A comparative performance evaluation of write barrier implementations. In Conference on Object Oriented Programming Systems, Languages, and Applications (OOPSLA '92). Vancouver, Canada, Oct. 1992

- [22] McCarthy, J., Recursive functions of symbolic expressions and their computation by machine. Communications of the ACM, Vol. 3, 4 (April 1960), 184-195

- [23] Moon, D. Garbage Collection in a Large LISP System. 1984 ACM Symposium on LISP and Functional Programming. (Aug. 1984) 235-246

- [24] Nilsen, K., Schmidt, W. Cost-effective object space management for hardware-assisted real-time garbage collection. ACM Letters on Programming Languages and Systems (LOPLAS), Vol. 1, 4 (Dec. 1992)

- [25] Ossia, Y., Ben-Yitzhak, O., Segal, M., Mostly concurrent compaction for mark-sweep GC. In Proceedings of the 2004 International Symposium on Memory Management, Vancouver, Canada, 2004.

- [26] Steele, G. Multiprocessing compactifying garbage collection. Communications of the ACM, Vol. 18, 9 (Sept. 1975), 495- 508

- [27] Suganuma, T., Ogasawara, T., Takeuchi, M., Yasue, T., Kawahito, M., Ishizaki, K., Komatsd, H., and Nakatani, T. Overview of the IBM Java Just-in-Time Compiler, IBM Systems Journal, 39(1), 2000.

- [28] Sun Microsystems. 2001. The Java HotSpot virtual machine. White paper. Sun Microsystems, Santa Clara, CA.

- [29] Williams, I., Wolczko, M. An Object-Based Memory Architecture. In Implementing Persistent Object Bases: Proceedings of the Fourth International Workshop on Persistent Object Systems, pages 114-130. Morgan Kaufmann Publishers, Inc., 1991.

- [30] Wilson, P. Uniprocessor Garbage Collection Techniques. In 1992 Proceedings of the International Workshop on Memory Management (IWMM 92). Saint-Malo (France), 1992

- [31] Wilson, P., Johnstone, M., Neely, M., Boles, D., Dynamic Storage Allocation: A Survey and Critical Review. In Proceedings of the International Workshop on Memory Management (IWMM 95), 1995 

> ������������� [`Azul-The-Pauseless-GC-Algorithm`](http://yuledanao.com/dl/Azul-The-Pauseless-GC-Algorithm.pdf)

����ʱ��: 2019��05��06��

������Ա: ��ê <https://renfufei.blog.csdn.net/>
