
![warp](https://user-images.githubusercontent.com/5663952/69431113-580f9500-0d37-11ea-8265-46142e7b15cc.png)

S3 benchmarking tool. 

# configuration

Warp can be configured either using commandline parameters or environment variables. 

The S3 server to use can be specified on the commandline using `-host`, `-access-key`, `-secret-key` and optionally `-tls` to specify TLS.

It is also possible to set the same parameters using the `WARP_HOST`, `WARP_ACCESS_KEY`, `WARP_SECRET_KEY` and `WARP_TLS` environment variables.

Multiple hosts can be specified as comma-separated values, for instance `10.0.0.1:9000,10.0.0.2:9000` 
will do a round-robin between the specified servers. 

Alternatively numerical ranges can be specified using `10.0.0.{1...10}:9000` which will add `10.0.0.1` through `10.0.0.10`.
This syntax can be used for any part of the host name and port.  
 
The credentials must be able to create, delete and list buckets and upload files and perform the operation requested.

By default operations are performed on a bucket called `warp-benchmark-bucket`.
This can be changed using the `-bucket` parameter. 
Do however note that the bucket will be completely cleaned before and after each run, 
so it should *not* contain any data.

If you are running TLS, you can enable server-side-encryption of objects using `-encrypt`. 
A random key will be generated and used.

# usage

`warp command [options]`

# benchmarks

All benchmarks operate concurrently. 
By default the processor determines the number of operations that will be running concurrently.
This can however also be tweaked using the `-concurrent` parameter.

Tweaking concurrency can have an impact on performance, especially if there is latency to the server tested.

Most benchmarks will also use different prefixes for each "thread" running.

By default all benchmarks save all request details to a file named `warp-operation-yyyy-mm-dd[hhmmss]-xxxx.csv.zst`.
A custom file name can be specified using the `-benchdata` parameter.
The raw data is [zstandard](https://facebook.github.io/zstd/) compressed CSV data.

When running benchmarks on several clients, it is possible to synchronize their start time 
using the `-syncstart` parameter.
The time format is 'hh:mm' where hours are specified in 24h format, and parsed as local server time.
Using this will make it more reliable to [merge benchmarks](https://github.com/minio/warp#merging-benchmarks) 
from the clients for total result. 
 
## get

Benchmarking get operations will upload `-objects` objects of size `-obj.size` and attempt to 
download as many it can within `-duration`. 

Objects will be uploaded with `-concurrent` different prefixes, except if `-noprefix` is specified. 
Downloads are chosen randomly between all uploaded data.

When downloading, the benchmark will attempt to run `-concurrent` concurrent downloads.

The analysis will include the upload stats as `PUT` operations and the `GET` operations.

```
Operation: GET
* Average: 2344.50 MB/s, 234.45 obj/s, 234.44 ops ended/s (59.119s)

Aggregated, split into 59 x 1s time segments:
* Fastest: 2693.83 MB/s, 269.38 obj/s, 269.00 ops ended/s (1s)
* 50% Median: 2419.56 MB/s, 241.96 obj/s, 240.00 ops ended/s (1s)
* Slowest: 1137.36 MB/s, 113.74 obj/s, 112.00 ops ended/s (1s)
```

The `GET` operations will contain the time until the first byte was received. 
This can be accessed using the `-requests` parameter.

## put

Benchmarking put operations will upload objects of size `-obj.size` until `-duration` time has elapsed. 

Objects will be uploaded with `-concurrent` different prefixes, except if `-noprefix` is specified. 

```
Operation: PUT
* Average: 971.75 MB/s, 97.18 obj/s, 97.16 ops ended/s (59.417s)

Aggregated, split into 59 x 1s time segments:
* Fastest: 1591.40 MB/s, 159.14 obj/s, 161.00 ops ended/s (1s)
* 50% Median: 919.79 MB/s, 91.98 obj/s, 95.00 ops ended/s (1s)
* Slowest: 347.95 MB/s, 34.80 obj/s, 32.00 ops ended/s (1s)
```

## delete

Benchmarking delete operations will upload `-objects` objects of size `-obj.size` and attempt to 
delete as many it can within `-duration`. 

The delete operations are done in `-batch` objects per request in `-concurrent` concurrently running requests.

If there are no more objects left the benchmark will end.  

The analysis will include the upload stats as `PUT` operations and the `DELETE` operations.

```
Operation: DELETE 100 objects per operation
* Average: 2520.27 obj/s, 25.03 ops ended/s (38.554s)

Aggregated, split into 38 x 1s time segments:
* Fastest: 2955.85 obj/s, 36.00 ops ended/s (1s)
* 50% Median: 2538.10 obj/s, 25.00 ops ended/s (1s)
* Slowest: 1919.86 obj/s, 23.00 ops ended/s (1s)
```

## list

Benchmarking list operations will upload `-objects` objects of size `-obj.size` with `-concurrent` prefixes.

The list operations are done per prefix.

The analysis will include the upload stats as `PUT` operations and the `LIST` operations separately.
The time from request start to first object is recorded as well and can be accessed using the `-requests` parameter.

```
Operation: LIST 833 objects per operation
* Average: 30991.05 obj/s, 37.10 ops ended/s (59.387s)

Aggregated, split into 59 x 1s time segments:
* Fastest: 31831.96 obj/s, 39.00 ops ended/s (1s)
* 50% Median: 31199.61 obj/s, 38.00 ops ended/s (1s)
* Slowest: 27917.33 obj/s, 35.00 ops ended/s (1s)
```

# Analysis

When benchmarks have finished all request data will be saved to a file and an analysis will be shown.

The saved data can be re-evaluated by running `warp analyze (filename)`. 

It is possible to merge analyses from concurrent runs using the `warp merge file1 file2 ...`.
This will combine the data as if it was run on the same client. 
Only the time segments that was actually overlapping will be considered.
This is based on the absolute time of each recording, 
so be sure that clocks are reasonably synchronized or use the `-syncstart` parameter.  

## Analysis data

All analysis will be done on a reduced part of the full data. 
The data aggregation will *start* when all threads have completed one request and 
the time segment will *stop* when the last request of a thread is initiated.

This is to exclude variations due to warm-up and threads finishing at different times.
Therefore the analysis time will typically be slightly below the selected benchmark duration.

In this run "only" 42.9 seconds are included in the aggregate data, due to big payload size and low throughput:
```
Operation: PUT
* Average: 37.19 MB/s, 0.37 obj/s, 0.33 ops ended/s (42.957s)
```

The benchmark run is then divided into fixed duration *segments* specified by `-analyze.dur`, default 1s. 
For each segment the throughput is calculated across all threads.

The analysis output will display the fastest, slowest and 50% median segment.

```
Aggregated, split into 59 x 1s time segments:
* Fastest: 2693.83 MB/s, 269.38 obj/s, 269.00 ops ended/s (1s)
* 50% Median: 2419.56 MB/s, 241.96 obj/s, 240.00 ops ended/s (1s)
* Slowest: 1137.36 MB/s, 113.74 obj/s, 112.00 ops ended/s (1s)
```
### Per request statistics

By adding the `-requests` parameter it is possible to display per request statistics.

This is not enabled by default, since it is assumed the benchmarks are throughput limited, 
but in certain scenarios it can be useful to determine problems with individual hosts for instance.

Example:

```
Operation: GET. Concurrency: 12. Hosts: 7.                                                       
                                                                                                 
Requests - 16720:                                                                                
 * Fastest: 2.9965ms Slowest: 62.9993ms 50%: 21.0006ms 90%: 31.0021ms 99%: 41.0016ms             
 * First Byte: Average: 20.575134ms, Median: 20.0007ms, Best: 1.9985ms, Worst: 62.9993ms         
                                                                                                 
Requests by host:                                                                                
 * http://127.0.0.1:9001 - 2395 requests:                                                        
        - Fastest: 2.9965ms Slowest: 55.0015ms 50%: 18.0002ms 90%: 28.001ms                      
        - First Byte: Average: 17.139147ms, Median: 16.9998ms, Best: 1.9985ms, Worst: 53.0026ms  
 * http://127.0.0.1:9002 - 2395 requests:                                                        
        - Fastest: 4.999ms Slowest: 60.9925ms 50%: 20.9993ms 90%: 31.001ms                       
        - First Byte: Average: 20.174683ms, Median: 19.9996ms, Best: 3.999ms, Worst: 59.9912ms   
 * http://127.0.0.1:9003 - 2395 requests:                                                        
        - Fastest: 6.9988ms Slowest: 56.0005ms 50%: 20.9978ms 90%: 31.001ms                      
        - First Byte: Average: 20.272876ms, Median: 19.9983ms, Best: 5.0012ms, Worst: 55.0012ms  
 * http://127.0.0.1:9004 - 2395 requests:                                                        
        - Fastest: 5.0002ms Slowest: 62.9993ms 50%: 22.0009ms 90%: 33.001ms                      
        - First Byte: Average: 22.039164ms, Median: 21.0015ms, Best: 4.0003ms, Worst: 62.9993ms  
 * http://127.0.0.1:9005 - 2396 requests:                                                        
        - Fastest: 6.9934ms Slowest: 54.002ms 50%: 21.0008ms 90%: 30.9998ms                      
        - First Byte: Average: 20.871833ms, Median: 20.0006ms, Best: 4.9998ms, Worst: 52.0019ms  
 * http://127.0.0.1:9006 - 2396 requests:                                                        
        - Fastest: 6.0019ms Slowest: 54.9972ms 50%: 22.9985ms 90%: 33.0007ms                     
        - First Byte: Average: 22.430863ms, Median: 21.9986ms, Best: 5.0008ms, Worst: 53.9981ms  
 * http://127.0.0.1:9007 - 2396 requests:                                                        
        - Fastest: 7.9968ms Slowest: 55.0899ms 50%: 21.998ms 90%: 30.9998ms                      
        - First Byte: Average: 21.049681ms, Median: 20.9989ms, Best: 6.9958ms, Worst: 54.0884ms  
```

The fastest and slowest request times are shown, as well as selected percentiles and the total amount is requests considered.

Note that different metrics are used to select the number of requests per host and for the combined, so there will likely be differences.

### CSV output

It is possible to output the CSV data of analysis using `-analyze.out=filename.csv` 
which will write the CSV data to the specified file. 

These are the data fields exported:

| Header              | Description |
|---------------------|-------------|
| `index`             | Index of the segment  |
| `op`                | Operation executed  |
| `duration_s`        | Duration of the segment in seconds  |
| `objects_per_op`    | Objects per operation  |
| `bytes`             | Total bytes of operations (*distributed)  |
| `full_ops`          | Operations completely contained within segment  |
| `partial_ops`       | Operations that either started or ended outside the segment, but was also executed during segment  |
| `ops_started`       | Operations started within segment  |
| `ops_ended`         | Operations ended within the segment  |
| `errors`            | Errors logged on operations ending within the segment  |
| `mb_per_sec`        | MB/s of operations within the segment (*distributed)  |
| `ops_ended_per_sec` | Operations that ended within the segment per second  |
| `objs_per_sec`      | Objects per second processed in the segment (*distributed)  |
| `start_time`        | Absolute start time of the segment  |
| `end_time`          | Absolute end time of the segment  |

Some of these fields are *distributed*. 
This means that the data of partial operations have been distributed across the segments they occur in.
The bigger a percentage of the operation is within a segment the larger part of it has been attributed there.

This is why there can be a partial object attributed to a segment, because only a part of the operation took place in the segment.

## Comparing Benchmarks

It is possible to compare two recorded runs using the `warp cmp (file-before) (file-after)` to
see the differences between before and after. 
There is no need for 'before' to be chronologically before 'after', but the differences will be shown
as change from 'before' to 'after'.

An example:
```
λ warp cmp warp-get-2019-11-29[125341]-7ylR.csv.zst warp-get-2019-11-29[124533]-HOhm.csv.zst                   
-------------------                                                                                            
Operation: PUT                                                                                                 
Duration: 1m4s -> 1m2s                                                                                         
* Average: +2.63% (+1.0 MB/s) throughput, +2.63% (+1.0) obj/s                                                  
* Fastest: -4.51% (-4.1) obj/s                                                                                 
* 50% Median: +3.11% (+1.1 MB/s) throughput, +3.11% (+1.1) obj/s                                               
* Slowest: +1.66% (+0.4 MB/s) throughput, +1.66% (+0.4) obj/s                                                  
-------------------                                                                                            
Operation: GET                                                                                                 
Operations: 16768 -> 171105                                                                                    
Duration: 30s -> 5m0s                                                                                          
* Average: +2.10% (+11.7 MB/s) throughput, +2.10% (+11.7) obj/s                                                
* First Byte: Average: -405.876µs (-2%), Median: -2.1µs (-0%), Best: -998.1µs (-50%), Worst: +41.0014ms (+65%) 
* Fastest: +2.35% (+14.0 MB/s) throughput, +2.35% (+14.0) obj/s                                                
* 50% Median: +2.81% (+15.8 MB/s) throughput, +2.81% (+15.8) obj/s                                             
* Slowest: -10.02% (-52.0) obj/s                                                                               
```

All relevant differences are listed. This is two `warp get` runs. 
Differences in parameters will be shown.

The usual analysis parameters can be applied to define segment lengths.

## Merging benchmarks

It is possible to merge runs from several clients using the `warp merge (file1) (file2) [additional files...]` command.

The command will output a combined data file with all data that overlap in time.

The combined output will effectively be the same as having run a single benchmark with a higher concurrency setting.
The main reason for running the benchmark on several clients would be to help eliminate client bottlenecks.

It is important to note that only data that strictly overlaps in absolute time will be considered for analysis.

When running benchmarks on several clients it is likely a good idea to specify the `-noclear` parameter so
clients don't accidentally delete each others data on startup or shutdown.

# Server Profiling

When running against a MinIO server it is possible to enable profiling while the benchmark is running.

This is done by adding `-serverprof=type` parameter with the type of profile you would like. This requires that the credentials allows admin access for the first host.

| Type  | Description                                                                                                                                |
|-------|--------------------------------------------------------------------------------------------------------------------------------------------|
| cpu   | CPU profile determines where a program spends its time while actively consuming CPU cycles (as opposed while sleeping or waiting for I/O). |
| mem   | Heap profile reports the currently live allocations; used to monitor current memory usage or check for memory leaks.                       |
| block | Block profile show where goroutines block waiting on synchronization primitives (including timer channels).                                |
| mutex | Mutex profile reports the lock contentions. When you think your CPU is not fully utilized due to a mutex contention, use this profile.     |
| trace | A detailed trace of execution of the current program. This will include information about goroutine scheduling and garbage collection.              |

Profiles for all cluster members will be downloaded as a zip file. Analyzing the profiles requires the Go tools to be installed. See [Profiling Go Programs](https://blog.golang.org/profiling-go-programs) for basic usage of the profile tools and an introduction to the [Go execution tracer](https://blog.gopheracademy.com/advent-2017/go-execution-tracer/) for more information.

