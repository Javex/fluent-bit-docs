# Buffering & Storage

The end-goal of [Fluent Bit](https://fluentbit.io) is to collect, parse, filter and ship logs to a central place. In this workflow there are many phases and one of the critical pieces is the ability to do _buffering_ : a mechanism to place processed data into a temporal location until is ready to be shipped.

By default when Fluent Bit process data, it uses Memory as a primary and temporary place to store the records, but there are certain scenarios where would be ideal to have a persistent buffering mechanism based in the filesystem to provide aggregation and data safety capabilities.

Choosing the right configuration is critical and the behavior of the service can be conditioned based in the backpressure settings. Before to jump into the configuration properties let's understand the relationship between _Chunks_, _Memory_, _Filesystem_ and _Backpressure_.

## Chunks, Memory, Filesystem and Backpressure

Understanding the chunks, buffering and backpressure concepts is critical for a proper configuration. Let's do a recap of the meaning of these concepts.

#### Chunks

When an input plugin \(source\) emit records, the engine group the records together in a _Chunk_. A Chunk size usually is around 2MB. By configuration, the engine decide where to place this Chunk, the default is that all chunks are created only in memory.

#### Buffering and Memory

As mentioned above, the Chunks generated by the engine are placed in memory but this is configurable.

If memory is the only mechanism set for the input plugin, it will just store data as much as it can there \(memory\). This is the fastest mechanism with less system overhead, but if the service is not able to deliver the records fast enough because of a slow network or an unresponsive remote service, Fluent Bit memory usage will increase since it will accumulate more data than it can deliver.

On a high load environment with backpressure the risks of having high memory usage is the chance to get killed by the Kernel \(OOM Killer\). A workaround for this backpressure scenario is to limit the amount of memory in records that an input plugin can register, this configuration property is called `mem_buf_limit`: if a plugin have enqueued more than `mem_buf_limit`, it won't be able to ingest more until it data can be delivered or flushed properly. On this scenario the input plugin in question is paused.

The workaround of `mem_buf_limit` is good for certain scenarios and environments, it helps to control the memory usage of the service, but at the costs that if a file gets rotated while paused, you might lose that data since it won't be able to register new records. This can happen with any input source plugin. The goal of `mem_buf_limit` is memory control and survival of the service.

For full data safety guarantee, use filesystem buffering.

#### Filesystem buffering to the rescue

Filesystem buffering enabled helps with backpressure and overall memory control.

Behind the scenes, Memory and Filesystem buffering mechanisms are **not** mutual exclusive, indeed when enabling filesystem buffering for your input plugin \(source\) you are getting the best of the two worlds: performance and data safety.

When the Filesystem buffering is enabled, the behavior of the engine is different, upon Chunk creation, it stores the content in memory but also it maps a copy on disk \(through [mmap\(2\)](https://man7.org/linux/man-pages/man2/mmap.2.html)\), this Chunk is active in memory and backed up in disk is called to be `up` which means "the chunk content is up in memory".

How this Filesystem buffering mechanism deals with high memory usage and backpressure ?: Fluent Bit controls the number of Chunks that are `up` in memory.

By default, the engine allows to have 128 Chunks `up` in memory in total \(considering all Chunks\), this value is controlled by service property `storage.max_chunks_up`. The active Chunks that are `up` are ready for delivery and the ones that still are receiving records. Any other remaining Chunk is in a `down` state, which means that's only in the filesystem and won't be `up` in memory unless is ready to be delivered.

If the input plugin has enabled `mem_buf_limit` and `storage.type` as `filesystem`, when reaching the `mem_buf_limit` threshold, instead of the plugin being paused, all new data will go to Chunks that are `down` in the filesystem. This allows to control the memory usage by the service but also providing a a guarantee that the service won't lose any data.

**Limiting Filesystem  space for Chunks**

Fluent Bit implements the concept of logical queues: a Chunk based on its Tag, can be routed to multiple destinations, so internally we keep a reference from where a Chunk was created and where it needs to go.

It's common to find cases that if we have multiple destinations for a Chunk, one of the destination might be slower than the other, and maybe one of the destinations is generating backpressure and not all of them. On this scenario how do we limit the amount of filesystem Chunks that we are logically queueing ?.

Starting from Fluent Bit v1.6, we introduced the new configuration property for output plugins called `storage.total_limit_size` which limits the number of Chunks that exists in the file system for a certain logical output destination. If one destinations reaches the `storage.total_limit_size` limit, the oldest Chunk from it queue for that logical output destination will be discarded.

## Configuration

The storage layer configuration takes place in three areas:

* Service Section
* Input Section
* Output Section

The known Service section configure a global environment for the storage layer, the Input sections defines which buffering mechanism to use and the output the limits for the logical queues.

### Service Section Configuration

The Service section refers to the section defined in the main [configuration file](configuring-fluent-bit/configuration-file.md):

| Key | Description | Default |
| :--- | :--- | :--- |
| storage.path | Set an optional location in the file system to store streams and chunks of data. If this parameter is not set, Input plugins can only use in-memory buffering. |  |
| storage.sync | Configure the synchronization mode used to store the data into the file system. It can take the values _normal_ or _full_. | normal |
| storage.checksum | Enable the data integrity check when writing and reading data from the filesystem. The storage layer uses the CRC32 algorithm. | Off |
| storage.max\_chunks\_up | If the input plugin has enabled `filesystem` storage type, this property sets the maximum number of Chunks that can be `up` in memory. This helps to control memory usage. | 128 |
| storage.backlog.mem\_limit | If _storage.path_ is set, Fluent Bit will look for data chunks that were not delivered and are still in the storage layer, these are called _backlog_ data. This option configure a hint of maximum value of memory to use when processing these records. | 5M |
| storage.metrics | If `http_server` option has been enable in the main `[SERVICE]` section, this option registers a new endpoint where internal metrics of the storage layer can be consumed. For more details refer to the [Monitoring](monitoring.md) section. | off |

a Service section will look like this:

```python
[SERVICE]
    flush                     1
    log_Level                 info
    storage.path              /var/log/flb-storage/
    storage.sync              normal
    storage.checksum          off
    storage.backlog.mem_limit 5M
```

that configuration configure an optional buffering mechanism where it root for data is _/var/log/flb-storage/_, it will use _normal_ synchronization mode, without checksum and up to a maximum of 5MB of memory when processing backlog data.

### Input Section Configuration

Optionally, any Input plugin can configure their storage preference, the following table describe the options available:

| Key | Description | Default |
| :--- | :--- | :--- |
| storage.type | Specify the buffering mechanism to use. It can be _memory_ or _filesystem_. | memory |

The following example configure a service that offers filesystem buffering capabilities and two Input plugins being the first based in filesystem and the second with memory only.

```python
[SERVICE]
    flush                     1
    log_Level                 info
    storage.path              /var/log/flb-storage/
    storage.sync              normal
    storage.checksum          off
    storage.backlog.mem_limit 5M

[INPUT]
    name          cpu
    storage.type  filesystem

[INPUT]
    name          mem
    storage.type  memory
```

### Output Section Configuration

If certain chunks are filesystem storage.type based, it's possible to control the size of the logical queue for an output plugin. The following table describe the options available:

| Key | Description | Default |
| :--- | :--- | :--- |
| storage.total\_limit\_size | Limit the maximum number of Chunks in the filesystem for the current output logical destination. |  |

The following example create records with CPU usage samples in the filesystem and then they are delivered to Google Stackdriver service limiting the logical queue \(buffering\) to 5M:

```text
[SERVICE]
    flush                     1
    log_Level                 info
    storage.path              /var/log/flb-storage/
    storage.sync              normal
    storage.checksum          off
    storage.backlog.mem_limit 5M

[INPUT]
    name                      cpu
    storage.type              filesystem 

[OUTPUT]
    name                      stackdriver
    match                     *
    storage.total_limit_size  5M
```

If for some reason Fluent Bit get's offline because of a network issue, it will continuing buffering CPU samples but just keeping a maximum of 5M of the newest data.
