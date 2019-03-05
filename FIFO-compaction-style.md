FIFO压缩风格是最简单的压缩策略。很适合用于保存不是那么重要的事件日志数据（例如查询日志）。他会周期性删除旧的数据，所以基本来说，他是一种TTL压缩风格。

在FIFO压缩里，所有的文件都在Level 0。当数据的总大小超过CompactionOptionsFIFO::max_table_files_size配置的大小时，我们删除最老的表文件。这意味着数据的写放大总是1（还有WAL的写放大）

目前，CompactRange函数只是强制触发压缩，然后如果有需要，删除旧的表文件。他忽略函数的参数（开始和结束key）

由于我们不会重写键值对，我们也不会对key执行压缩过滤器方法。

请小心使用FIFO压缩风格。与其他压缩风格不同，他可能在不通知用户的情况下删除数据。

# 压缩

FIFO压缩可能会导致大量L0文件。查询可能会变得很慢，因为最坏情况下，我们可能需要搜索所有的这些文件。即使是bloom过滤器也可能无法得到一个好的性能。加入有1%的假阳性结果，1000个L0文件平均会导致10个假阳性结果，然后在最坏情况下，每个查询会生成10个IO。用户可以选择使用更多的bloom位来减少假阳性结果，但是他们需要为此付出更多的内存。在某些情况下，bloom过滤器检查的CPU开支可能会过高。

为了解决这个问题，用户可以选择允许一些轻量压缩发生。这可能会让写IO变成两倍，但是可以显著减少L0的文件。某些时候对于用户来说是合理的权衡。

这个功能在5.5版本中引入。用户可以通过CompactionOptionsFIFO.allow_compaction = true来打开这个功能。他会尝试选择至少level0_file_num_compaction_trigger个从memtable落盘的文件，然后合并他们。

特别的，我们总是从最新的level0_file_num_compaction_trigger文件开始，尝试包含尽可能多的文件进行压缩。我们使用 total_compaction_size / (number_files_in_compaction - 1) 计算已经压缩的每个文件的大小。我们总是以保证这个数字最小，并且不多于options.write_buffer_size，这两个条件来挑选文件。在一个典型的工作场景，他总会压缩level0_file_num_compaction_trigger个刚落盘的文件。

例如，如果level0_file_num_compaction_trigger = 8，每个罗盘文件为100MB。那么只要达到了8个文件，他们会被压缩为一个800MB的文件。然后等我们有了8个新的100MB文件，他们会被压缩成第二个800MB的文件，以此类推。最终我们有一系列800MB，但是不超过8100MB的文件。

请注意，由于最老的文件被压缩了，FIFO删除的文件也变大了，所以可能排序好的数据会比没有压缩的数据略微少一点。

## FIFO带TTL压缩

一个新的，名为FIFO带TTL压缩的功能在RocksDB5.7被引入。

目前，FIFO压缩目前只考虑文件总大小，比如：如果db的大小超过compaction_options_fifo.max_table_files_size，丢弃最老的文件，一个个地删除知道总大小小于阈值。有时候，生产环境上打开这个，会随着有机增长把生产环境搞乱。

一个新的选项，compaction_options_fifo.ttl，被引入，用来删除超过TTL的SST文件。这个功能允许用户根据时间丢弃文件而不总是根据大小来，比如说，丢弃所有一周前或者一个月前的数据。

限制：

- 这个选项目前只在max_open_files为-1时，对基于块的表格式使用。
- FIFO带TTL仍旧在配置的大小范围内工作，比如说，如果观察到TTL无法让文件总数量少于配置的大小，RocksDB会暂时下降到基于大小的FIFO删除。


