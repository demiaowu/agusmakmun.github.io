---
layout: post
title:  Leveldb 之 Compaction
date:   2018-03-22 12:39:12 +0700
categories: [LSM-Tree]
---

* Kramdown table of content
{:toc, toc}

Backgroud会开启一个线程来处理compact操作。所有的leveldb调用的compact操作都会插入这个线程的任务队列中等待处理。
Compaction有两种，分别是：
- Minor Compaction，指的是memtable持久化为sst文件。
- Major Compaction，指的是sst文件之间的compaction。
在Major Compaction主要有三种分别为：
a)	Manual Compaction，是人工触发的Compaction，由外部接口调用产生，例如在ceph调用的Compaction都是Manual Compaction，实际其内部触发调用的接口是：
void DBImpl::CompactRange(const Slice* begin, const Slice* end)
b)	Size Compaction，是根据每个level的总文件大小来触发，注意Size Compation的优先级高于Seek Compaction，具体描述参见Notes 2；
c)	Seek Compaction，每个文件的seek次数都有一个阈值，如果超过了这个阈值，那么认为这个文件需要Compact。

  
详细的内容参考写的文档**[LevelDB compaction]**(http://note.youdao.com/noteshare?id=eadbb414fb9267f9949463691c20feb2&sub=6A5102A3670D4DD48AE51201EAA7DB15)

----------------

