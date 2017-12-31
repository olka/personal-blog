---
title: "node.js performance on Raspberry Pi and Tinker Board"
date: 2017-12-30T17:43:58+02:00
draft: false
---
Raspberry Pi vs Tinker Board: node.js performance
====================================================

Preface
----------------
--------------------------------------------
Recently I've bought both Raspberry Pi and Tinker Board to experiment with microservices performance, cause in the cloud you never know how busy your host is. 
As an application under test I've found just perfect match - [Dockerised Microservices Calculator](https://github.com/cloudacademy/aws-xray-microservices-calc) by [Jeremy Cook](https://github.com/jeremycook123) written on node.js. It's dockerised microservices calculator that has been designed to evaluate simple and complex mathematical expressions with any valid user provided expression using the order of operations (or operator precedence) rules. You should definitely check it out if your're interested in playing with microservices.

Node.js installation process is the same for both platforms(since they're just Debian forks) and as simple as executing two lines in the shell:
```bash
curl -sL https://deb.nodesource.com/setup_9.x | sudo -E bash -
sudo apt-get install -y nodejs
```
Unfortunately node for ARM architecture compiled with frame pointers omission so there are problems with [flame graphs](http://www.brendangregg.com/flamegraphs.html) plotting: regular perf output provides no information to [Brendan's](http://www.brendangregg.com) scripts and 
```bash
perf record -F 99 -p {$PID} -g --call-graph dwarf
``` 
generates huge amount of data (with disk I/O oprations) which drastically decrease performance of Raspberry. In addition those dwarf traces can't be properly handled by [FlameGraph](https://github.com/brendangregg/FlameGraph) scripts - everything except kernel calls is just flat so overall efficiency of flame graphs on ARM is somewhat questionable. 

But even with those limitations there're some interesting findings worth sharing.

Comparison
----------------
--------------------------------------------
Load tests were performed with [Gatling](http://gatling.io) testing tool and overall approach was pretty straightforward:

1. [Tune OS](https://gatling.io/docs/2.3/general/operations/)
2. Start [node exporter](https://github.com/prometheus/node_exporter) and configure Prometheus with Grafana
3. Execute healthcheck load test 
4. Check saturation point and monitoring results
5. Execute calculator load test which evaluates expression (2*(9+22/5)-((9-1)/4)^2)+(3^2+((5*5-1)/2))
6. Check saturation point and monitoring results
7. Report analysis

### Saturation points
#### Healthcheck test
Raspberry Pi:
<img src="./raspberry_rps.png" width="100%"/>
Tinker Board:
<img src="./tinker_rps.png" width="100%"/>
On simple healthcheck test Tinker Board is more than 2x times faster with ~3000 RPS while Raspberry have only ~1350 RPS

#### Calculator test
Raspberry Pi:
<img src="./raspberry_calc_rps.png" width="100%">
Tinker Board:
<img src="./tinker_calc_rps.png" width="100%"/>
For calculator test we can see similar picture: Pi with ~65 RPS vs ~130 RPS of Tinker Board

#### Monitoring info
Raspberry Pi:
<img src="./rasp_monit.png" width="100%"/>
Here we have an evidence that it's really hard to get something more than ~6 MB/s as many [users](https://raspberrypi.stackexchange.com/questions/43330/raspberry-pi-usb-bus-speed) [reported](https://raspberrypi.stackexchange.com/questions/46076/soc-cpu-and-ethernet-controller-internal-connection-in-raspberry-pi-3)

Tinker Board:
<img src="./tinker_monit.png" width="100%"/>

#### Flame graphs
As I previously mentioned flame graphs not so usefull for this case but something we can see here:

Raspberry Pi:
<img src="./rasp_fg.svg" width="100%"/>
Tinker Board:
<img src="./tinker_fg.svg" width="100%"/>
Tinker Board can handle context switches a lot faster than Raspberry Pi but since dwarf adds huge amount of overhead and overall throughput of Tinker Board is higher - this information is not something that we can rely on.


### Conclusion
1. Overall Tinker Board 2x faster than Pi
2. Tinker Board can handle twice as more Context switches as Pi can
3. Tinker OS poorly optimised for disk IO which obviously slows everything down and drasctically decrease SD card lifetime.


### P.S.: Useful commands
Stats on system calls and their execution time  
```bash
strace -p {$PID} -c -w
```
Interrupts monitoring
```bash
watch -tdn1 cat /proc/interrupts
itop
```
Perf profiling without sudo privileges: 
```bash
sudo sysctl kernel.perf_event_paranoid=0
sudo sysctl kernel.kptr_restrict=0
```