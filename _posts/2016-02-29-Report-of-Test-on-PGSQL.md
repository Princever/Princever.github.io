---  
layout: post
title: Test of Parallel Sequence Scan
category: "Database"
---  

### Abstract ###

It is a report about a Test of Parallel Seq Scan on PostgreSQL. We tested the performance when number of worker threads is 0, 1, 2, 4, 6, 8, 12, 16, 20, 24. Then we visualize the result with perf and flame graph.





### Test design

The design of test is simple. We run the test on a 12-core server and record the running time and workload.

### Hardware spec and configuration ###

	OS					CentOS6.6-64
	RAM					8x8GB Micron 8GB DDR4 1Rx4
	Processor			2.4GHz Intel Xeon-Haswell (E5-2620-V3-HexCore)
	Processor			2.4GHz Intel Xeon-Haswell (E5-2620-V3-HexCore)
	Motherboard			SuperMicro X10DRU-i+
	Remote Mgmt 		Card	Aspeed AST2400 - Onboard
	Network Card		SuperMicro AOC-UR-i4XT
	Backplane			SuperMicro BPN-SAS-815TQ
	Power Supply		SuperMicro PWS-751P-1R
	Driver controller	Mainboard Onboard
	Security Device		SuperMicro AOM-TPM-9655V

### Software version, modification and configuration ###

	PostgreSQL			version 9.6devel
	perf				version 2.6.32-573.18.1.el6.x86_64.debug

### Schema design, data size and tools ###

1. We tested parallel seq scan with pgbench. With the scale factor of 2000 and 5000 whose cooresponding data size is 3200k lines and 8000k lines.

2. Setting `max_parallel_degree` to 0, 1, 2, 4, 6, 8, 12, 16, 20, 24, we ran the **SELECT** query and collocted the running time and detials of each workers.

3. Using perf to record the function calls of the process.

4. Draw the results in flame graph.

### Test queries ###

We use `EXPLAIN (ANALYZE true, VERBOSE true, BUFFERS true) select * from pgbench_accounts where filler = 'foo';` as the query.

### Results ###

#### Data = 3200k lines ####

<table class="table table-bordered table-striped table-condensed">
   <tr>
      <td>number of workers</td>
      <td>running time(s)</td>
      <td>efficiency</td>
   </tr>
   <tr>
      <td>no-para</td>
      <td>117,253 </td>
      <td>1.00 </td>
   </tr>
   <tr>
      <td>1</td>
      <td>59,796 </td>
      <td>1.96 </td>
   </tr>
   <tr>
      <td>2</td>
      <td>39,986 </td>
      <td>2.93 </td>
   </tr>
   <tr>
      <td>4</td>
      <td>24,152 </td>
      <td>4.85 </td>
   </tr>
   <tr>
      <td>6</td>
      <td>17,274 </td>
      <td>6.79 </td>
   </tr>
   <tr>
      <td>8</td>
      <td>13,470 </td>
      <td>8.70 </td>
   </tr>
   <tr>
      <td>12</td>
      <td>9,858 </td>
      <td>11.89 </td>
   </tr>
   <tr>
      <td>16</td>
      <td>8,868 </td>
      <td>13.22 </td>
   </tr>
   <tr>
      <td>20</td>
      <td>8,060 </td>
      <td>14.55 </td>
   </tr>
   <tr>
      <td>24</td>
      <td>7,848 </td>
      <td>14.94 </td>
   </tr>
</table>


#### Data = 8000k lines ####

<table class="table table-bordered table-striped table-condensed">
   <tr>
      <td>number of workers</td>
      <td>running time(s)</td>
      <td>efficiency</td>
   </tr>
   <tr>
      <td>1</td>
      <td>221,332 </td>
      <td>1.00 </td>
   </tr>
   <tr>
      <td>2</td>
      <td>183,937 </td>
      <td>1.20 </td>
   </tr>
   <tr>
      <td>4</td>
      <td>197,698 </td>
      <td>1.12 </td>
   </tr>
   <tr>
      <td>6</td>
      <td>165,121 </td>
      <td>1.34 </td>
   </tr>
   <tr>
      <td>8</td>
      <td>197,104 </td>
      <td>1.12 </td>
   </tr>
</table>


### Conclusion and Future work ###

![]({{ site.baseurl }}/res/images/reportofpgsql/1.png)
![]({{ site.baseurl }}/res/images/reportofpgsql/2.png)

### Refrences ###

### Appendix. ###