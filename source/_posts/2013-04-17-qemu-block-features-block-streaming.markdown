---
comments: true
date: 2013-04-18 10:30:31
layout: post
slug: live-snapshot-merge
title: 'QEMU Block Fetures: live snapshot merge (block commit/block stream)'
categories:
- QEMU
- Virtualization
tags: [qemu, kvm, libvirt]

---

### Block Streaming

1. Ref: [QEMU-Feature-LiveSnapshotMerge-BlockStreaming](http://wiki.qemu.org/Features/Snapshots#Block_Streaming)
2. Process: Take data from parent image(s), and copy (stream) the data to the active layer.
3. QMP Steps
    1. Into Qemu and Prepare, [ref](#qmp_prepare)
    2. [QMP_INPUT]
        * Run Block Stream:
      <pre><code>
    	{ "execute": "block-stream",
    	     "arguments": 
    	     {"device":"drive0", "base": "win2003.qcow2", "speed": 20971520} 
    	}</code></pre>
	* CMD Args:
	    * device: block device to stream
	    * base: base image, only sectors above this image are streamed. optional
	    * speed: speed throttling, in MB/s, of the stream opt. optional
    3. [QMP_OUTPUT] After Job Completed, can see the QMP's output:
	<pre><code>{"timestamp": {"seconds": 1366281580, "microseconds": 788190}, "event": "BLOCK_JOB_COMPLETED", "data": {"device": "drive0", "len": 8589934592, "offset": 8589934592, "speed": 2097152, "type": "stream"}}</code></pre>
    4. Job Control / Job Query, [ref](#qmp_block_job_ctl)
4. HMP Steps 
    1. Into Qemu and Prepare, [ref](#hmp_prepare)
    2. [HMP_INPUT]
	* Do the snapshot:
	<pre><code>snapshot_blkdev drive0 /pkgs/imgs/win.snap</code></pre>
	* Run Block Stream:
	<pre><code>block_stream drive0 10 # 10 MB/s</code></pre>
    3. Job Control / Job Query, [ref](#hmp_block_job_ctl)
    4. concurrency-opts
	* block_stream again when block_stream is not finished yet:
	    * [HMP_OUTPUT]: <pre><code>Device 'drive0' is in use</code></pre>
	* another block_job, eg. block_commit when block_stream is not finished yet:
	    * [HMP_INPUT]: <pre><code>commit drive0</code></pre>
	    * [HMP_OUTPUT]: <pre><code>'commit' error for 'drive0': Device or resource busy</code></pre>
5. Libvirt Supprot
    * @see [TASK] IO mirror

### <a id="qmp_prepare"></a>QMP Prepare
1. [CMD] qemu [...] -qmp tcp:localhost:4444,server
    * Start QMP on a TCP socket, so that telnet can be used
    * -qmp conflicts with "-monitor"
2. [CMD] telnet localhost 4444
    * Run telnet
    * Should see the QMP's greeting banner:
    	<pre><code>
    	{"QMP":{"version": {"qemu": {"micro": 50, "minor": 13, "major": 0}, "package": ""}, "capabilities": []}} </code></pre>
3. [QMP_INPUT] 
    * Make QMP enter command mode:
    	<pre><code>
    	{"execute": "qmp_capabilities"}</code></pre>
4. [QMP_INPUT]
    * Query the device info:
    	<pre><code>
    	{"execute":"query-blockstats"} </code></pre>
    	or
    	<pre><code>
    	{ "execute": "human-monitor-command", "arguments": { "command-line": "info block" } } </code></pre>
    * Get the device info from the output

### <a id="hmp_prepare"></a> HMP Prepare
1. [CMD] qemu [...] -monitor stdio
2. [HMP_INPUT] Query the device info:
    	<pre><code>info block</code></pre>

### <a id="qmp_block_job_ctl"></a> QMP Block Job Control
1. [QMP_INPUT] query block info
	<pre><code>{ "execute": "query-block-jobs", "arguments": {} }</code></pre>
2. [QMP_INPUT] set block job speed
	<pre><code>{ "execute": "block-job-set-speed", "arguments": 
	    {"device":drive0,"speed":2097152} 
	}</code></pre>
3. [QMP_INPUT] cancel block job
	<pre><code>{ "execute": "block-job-cancel", "arguments": 
	    {"device":drive0,"force":1} 
	}</code></pre>
4. [QMP_INPUT] pause block job
	<pre><code>{ "execute": "block-job-pause", "arguments": 
	    {"device":drive0} 
	}</code></pre>
5. [QMP_INPUT] resume block job
	<pre><code>{ "execute": "block-job-resume", "arguments": 
	    {"device":drive0} 
	}</code></pre>
6. [QMP_INPUT] complete block job
	<pre><code>{ "execute": "block-job-complete", "arguments": 
	    {"device":drive0} 
	}</code></pre>

### <a id="hmp_block_job_ctl"></a> HMP Block Job Control
1. [HMP_INPUT] query block info
	<pre><code>info block-jobs</code></pre>
2. [HMP_INPUT] set block job speed
	<pre><code>block_job_set_speed drive0 20</code></pre>
3. [HMP_INPUT] cancel block job
	<pre><code>block_job_cancel drive0</code></pre>
4. [HMP_INPUT] pause block job
	<pre><code>block_job_pause drive0</code></pre>
5. [HMP_INPUT] resume block job
	<pre><code>block_job_resume drive0</code></pre>
6. [HMP_INPUT] complete block job
	<pre><code>block_job_complete drive0</code></pre>
