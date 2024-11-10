当前笔记参考的作者是dgpp_programer，主要是他的[写性能最强的kv数据库RocksDB全集详解](https://www.bilibili.com/video/BV1vDWseEEys/?spm_id_from=333.999.0.0)系列

本文章同步到我的笔记：[https://github.com/HentaiYang/NoteBooks](https://github.com/HentaiYang/NoteBooks)

---

**学习笔记：《RocksDB学习笔记》索引**
[RocksDB学习笔记#1 基本概念和简单使用](https://blog.csdn.net/qq_38876396/article/details/143467285)
[RocksDB学习笔记#2 SST、列族、Version、布隆过滤器](https://blog.csdn.net/qq_38876396/article/details/143469050)
[RocksDB学习笔记#3 写流程](https://blog.csdn.net/qq_38876396/article/details/143469414)
[RocksDB学习笔记#4 读流程](https://blog.csdn.net/qq_38876396/article/details/143469892)
[RocksDB学习笔记#5 Flush流程](https://blog.csdn.net/qq_38876396/article/details/143470573)
[RocksDB学习笔记#6 Compaction流程(1) —— 触发流程](https://blog.csdn.net/qq_38876396/article/details/143475637)
[RocksDB学习笔记#7 Compaction流程(2) —— 执行流程](https://blog.csdn.net/qq_38876396/article/details/143478199)

---

## 目录
* [1.读流程简介](#p1)
* [2.读流程源码解析](#p2)
* &nbsp;&nbsp;[2.1.Get()](#p21)
* &nbsp;&nbsp;[2.2.GetImpl()](#p22)
* &nbsp;&nbsp;&nbsp;&nbsp;[2.2.1.读memtable](#221)
* &nbsp;&nbsp;&nbsp;&nbsp;[2.2.2.读immemtable](#222)
* &nbsp;&nbsp;&nbsp;&nbsp;[2.2.3.读sst](#223)

---

# 1.读流程简介<a id="p1"></a>
按顺序读memtable、immemtable、sst，任何一个地方读到就返回。读sst文件时，L0层需要遍历所有sst文件，L1开始就采用二分方法读。

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/rocksdb/4/1.jpg"></div> 

# 2.读流程源码解析<a id="p2"></a>

## 2.1.Get()<a id="p21"></a>

与Put相似，Get主要调用的也是4个API，最终都会调用到DBImpl::Get方法：

```cpp
// 与Put相似，Get也根据是否指定列族、是否传入时间戳分为4个API
// 不指定列族的会用默认列族调用指定列族的API
virtual inline Status Get(const ReadOptions& options,
                          ColumnFamilyHandle* column_family, const Slice& key,
                          std::string* value) {
  PinnableSlice pinnable_val(value);
  auto s = Get(options, column_family, key, &pinnable_val);
  if (s.ok() && pinnable_val.IsPinned()) {
    value->assign(pinnable_val.data(), pinnable_val.size());
  }  // else value is already assigned
  return s;
}

virtual Status Get(const ReadOptions& options, const Slice& key,
                   std::string* value) {
  return Get(options, DefaultColumnFamily(), key, value);
}
 
virtual inline Status Get(const ReadOptions& options,
                          ColumnFamilyHandle* column_family, const Slice& key,
                          std::string* value, std::string* timestamp) {
  PinnableSlice pinnable_val(value);
  auto s = Get(options, column_family, key, &pinnable_val, timestamp);
  if (s.ok() && pinnable_val.IsPinned()) {
    value->assign(pinnable_val.data(), pinnable_val.size());
  }  // else value is already assigned
  return s;
}

virtual Status Get(const ReadOptions& options, const Slice& key,
                   std::string* value, std::string* timestamp) {
  return Get(options, DefaultColumnFamily(), key, value, timestamp);
}

// 不带时间戳的会调用这个，再去调用带时间戳的DBImpl::Get
Status DBImpl::Get(const ReadOptions& read_options,
                   ColumnFamilyHandle* column_family, const Slice& key,
                   PinnableSlice* value) {
  return Get(read_options, column_family, key, value, /*timestamp=*/nullptr);
}

// 带时间戳，最后一层Get
Status DBImpl::Get(const ReadOptions& _read_options,
                   ColumnFamilyHandle* column_family, const Slice& key,
                   PinnableSlice* value, std::string* timestamp) {
  value->Reset();
  ...
  ReadOptions read_options(_read_options);
  if (read_options.io_activity == Env::IOActivity::kUnknown) {
    read_options.io_activity = Env::IOActivity::kGet;
  }
  Status s = GetImpl(read_options, column_family, key, value, timestamp);
  return s;
}
```

## 2.2.GetImpl()<a id="p22"></a>
DBImpl::GetImpl有2个接口，最终调用的都是4个参数的接口，其负责统计性能、获取最新SuperVersion、读MemTable、读ImmemTable、读sst文件。

这三次读取分别调用MemTable::Get，MemTableListVersion::Get和Version::Get。
```cpp
// 第一层GetImpl，接收5个参数
Status DBImpl::GetImpl(const ReadOptions& read_options,
                       ColumnFamilyHandle* column_family, const Slice& key,
                       PinnableSlice* value, std::string* timestamp) {
  GetImplOptions get_impl_options;
  get_impl_options.column_family = column_family;
  get_impl_options.value = value;
  get_impl_options.timestamp = timestamp;

  Status s = GetImpl(read_options, key, get_impl_options);
  return s;
}

Status DBImpl::GetImpl(const ReadOptions& read_options, const Slice& key,
                       GetImplOptions& get_impl_options) {
  // 性能统计 ...
  StopWatch sw(immutable_db_options_.clock, stats_, DB_GET);
  PERF_TIMER_GUARD(get_snapshot_time);
  auto cfh = static_cast_with_check<ColumnFamilyHandleImpl>(
      get_impl_options.column_family);
  auto cfd = cfh->cfd();
  ...
  // 获取最新SuperVersion并+1计数
  SuperVersion* sv = GetAndRefSuperVersion(cfd);
  ...
  // Get默认不带snapshot，通过version_获取最新的seq给snapshot去读
  SequenceNumber snapshot;
  if (read_options.snapshot != nullptr) {
    if (get_impl_options.callback) {
      snapshot = get_impl_options.callback->max_visible_seq();
    } else {
      snapshot =
          reinterpret_cast<const SnapshotImpl*>(read_options.snapshot)->number_;
    }
  } else {
    snapshot = GetLastPublishedSequence();
if (get_impl_options.callback) {
  ...
    }
  }
  // 时间戳相关 ...
  // Prepare to store a list of merge operations if merge occurs.
  MergeContext merge_context;
  SequenceNumber max_covering_tombstone_seq = 0;

  Status s;
  // 要查找的key
  LookupKey lkey(key, snapshot, read_options.timestamp);
  PERF_TIMER_STOP(get_snapshot_time);

  bool skip_memtable = (read_options.read_tier == kPersistedTier &&
                        has_unpersisted_data_.load(std::memory_order_relaxed));
  bool done = false;
  std::string* timestamp =
      ucmp->timestamp_size() > 0 ? get_impl_options.timestamp : nullptr;
  // read流程：memtable -> immemtable -> 缓存和sst
  if (!skip_memtable) {
if (get_impl_options.get_value) {
  // 1.读memtable
      if (sv->mem->Get(
              lkey,
              get_impl_options.value ? get_impl_options.value->GetSelf()
                                     : nullptr,
              get_impl_options.columns, timestamp, &s, &merge_context,
              &max_covering_tombstone_seq, read_options,
              false /* immutable_memtable */, get_impl_options.callback,
              get_impl_options.is_blob_index)) {
        done = true;
        if (get_impl_options.value) {
          get_impl_options.value->PinSelf();
        }
        RecordTick(stats_, MEMTABLE_HIT);
        // 2.读immemtable
      } else if ((s.ok() || s.IsMergeInProgress()) &&
                 sv->imm->Get(lkey,
                              get_impl_options.value
                                  ? get_impl_options.value->GetSelf()
                                  : nullptr,
                              get_impl_options.columns, timestamp, &s,
                              &merge_context, &max_covering_tombstone_seq,
                              read_options, get_impl_options.callback,
                              get_impl_options.is_blob_index)) {
        done = true;
        if (get_impl_options.value) {
          get_impl_options.value->PinSelf();
        }
        RecordTick(stats_, MEMTABLE_HIT);
      }
    } else {
      // merge操作将结果暂时缓存在merge_context中
      if (sv->mem->Get(lkey, /*value=*/nullptr, /*columns=*/nullptr,
                       /*timestamp=*/nullptr, &s, &merge_context,
                       &max_covering_tombstone_seq, read_options,
                       false /* immutable_memtable */, nullptr, nullptr,
                       false)) {
        done = true;
        RecordTick(stats_, MEMTABLE_HIT);
      } else if ((s.ok() || s.IsMergeInProgress()) &&
                 sv->imm->GetMergeOperands(lkey, &s, &merge_context,
                                           &max_covering_tombstone_seq,
                                           read_options)) {
        done = true;
        RecordTick(stats_, MEMTABLE_HIT);
      }
    }
    if (!done && !s.ok() && !s.IsMergeInProgress()) {
      ReturnAndCleanupSuperVersion(cfd, sv);
      return s;
    }
  }
  PinnedIteratorsManager pinned_iters_mgr;
  // 3.读sst
  if (!done) {
    PERF_TIMER_GUARD(get_from_output_files_time);
    sv->current->Get(
        read_options, lkey, get_impl_options.value, get_impl_options.columns,
        timestamp, &s, &merge_context, &max_covering_tombstone_seq,
        &pinned_iters_mgr,
        get_impl_options.get_value ? get_impl_options.value_found : nullptr,
        nullptr, nullptr,
        get_impl_options.get_value ? get_impl_options.callback : nullptr,
        get_impl_options.get_value ? get_impl_options.is_blob_index : nullptr,
        get_impl_options.get_value);
    RecordTick(stats_, MEMTABLE_MISS);
  }
  // 读取到之后的操作，主要处理Merge操作的读取
  // 引用管理和清理
  // 记录性能和统计信息
  return s;
}
```

### 2.2.1.读memtable<a id="p221"></a>
首先，读取MemTable，使用接口为MemTable::Get，其先通过布隆过滤器快速判断是否有该key，而后通过GetFromTable读取：

```cpp
bool MemTable::Get(一大堆参数) {
  ...
  bool found_final_value = false;
  bool merge_in_progress = s->IsMergeInProgress();
  bool may_contain = true;
  Slice user_key_without_ts = StripTimestampFromUserKey(key.user_key(), ts_sz_);
  bool bloom_checked = false;
  if (bloom_filter_) {
    // 通过布隆过滤器过滤一次
    if (moptions_.memtable_whole_key_filtering) {
      may_contain = bloom_filter_->MayContain(user_key_without_ts);
      bloom_checked = true;
    } else {
      assert(prefix_extractor_);
      if (prefix_extractor_->InDomain(user_key_without_ts)) {
        may_contain = bloom_filter_->MayContain(
            prefix_extractor_->Transform(user_key_without_ts));
        bloom_checked = true;
      }
    }
  }

  if (bloom_filter_ && !may_contain) {
    PERF_COUNTER_ADD(bloom_memtable_miss_count, 1);
    *seq = kMaxSequenceNumber;
  } else {
    if (bloom_checked) {
      PERF_COUNTER_ADD(bloom_memtable_hit_count, 1);
    }
    // 通过GetFromTable读取
    GetFromTable(key, *max_covering_tombstone_seq, do_merge, callback,
                 is_blob_index, value, columns, timestamp, s, merge_context,
                 seq, &found_final_value, &merge_in_progress);
  }
  if (!found_final_value && merge_in_progress && !s->IsCorruption()) {
    *s = Status::MergeInProgress();
  }
  PERF_COUNTER_ADD(get_from_memtable_count, 1);
  return found_final_value;
}
```

GetFromTable会调用MemTableRep::Get()方法，主要通过callback_func（SaveValue）回传key和value：

```cpp
void MemTable::GetFromTable(一大堆参数) {
  // 进行一些变量赋值后，用MemTableRep去读
  table_->Get(key, &saver, SaveValue);
  *seq = saver.seq;
}

void MemTableRep::Get(const LookupKey& k, void* callback_args,
                      bool (*callback_func)(void* arg, const char* entry)) {
  // 获取一个迭代器，通过这个迭代器去skiplist查询对应的key
  auto iter = GetDynamicPrefixIterator();
  // 如果能在skiplist中查询到对应的key，就进入回调callback_func函数
  // 这里的回调为SaveValue函数，从iter->key()中提取key和value
  for (iter->Seek(k.internal_key(), k.memtable_key().data());
       iter->Valid() && callback_func(callback_args, iter->key());
       iter->Next()) {
  }
}
```

### 2.2.2.读immemtable<a id="p222"></a>

在memtable中没有找到时，到immemtable中查找，通过MemTableListVersion::Get调用MemTableListVersion::GetFromList()，其遍历immemtable list，然后读取数据，过程和读memtable差不多

```cpp
// 2.读immemtable
bool MemTableListVersion::Get(一大堆参数) {
  return GetFromList(&memlist_, key, value, columns, timestamp, s,
                     merge_context, max_covering_tombstone_seq, seq, read_opts,
                     callback, is_blob_index);
}
bool MemTableListVersion::GetFromList(一大堆参数) {
  *seq = kMaxSequenceNumber;
  // 遍历immemtable list，然后读取数据，流程同memtable
  for (auto& memtable : *list) {
    assert(memtable->IsFragmentedRangeTombstonesConstructed());
SequenceNumber current_seq = kMaxSequenceNumber;

    bool done =
        memtable->Get(key, value, columns, timestamp, s, merge_context,
                      max_covering_tombstone_seq, &current_seq, read_opts,
                      true /* immutable_memtable */, callback, is_blob_index);
    if (*seq == kMaxSequenceNumber) {
      *seq = current_seq;
    }

    if (done) {
      assert(*seq != kMaxSequenceNumber || s->IsNotFound());
      return true;
    }
    if (!s->ok() && !s->IsMergeInProgress() && !s->IsNotFound()) {
      return false;
    }
  }
  return false;
}
```

### 2.2.3.读sst<a id="p223"></a>

读sst通过接口Version::Get()，通过FilePicker来选取SST进行查询，对L0全部查询，而L1及以上二分查询，其选取逻辑在FilePicker::GetNextFile()中。

```cpp
// 3.读sst
void Version::Get(一大堆参数) {
  ...
  // 创建一个查询上下文
  GetContext get_context(
      user_comparator(), merge_operator_, info_log_, db_statistics_,
      status->ok() ? GetContext::kNotFound : GetContext::kMerge, user_key,
      do_merge ? value : nullptr, do_merge ? columns : nullptr,
      do_merge ? timestamp : nullptr, value_found, merge_context, do_merge,
      max_covering_tombstone_seq, clock_, seq,
      merge_operator_ ? pinned_iters_mgr : nullptr, callback, is_blob_to_use,
      tracing_get_id, &blob_fetcher);
  ...
  // 创建FilePicker，来选取SST进行查询
  // 该version的所有SST都存放在storage_info_.files_内
  FilePicker fp(user_key, ikey, &storage_info_.level_files_brief_,
                storage_info_.num_non_empty_levels_,
                &storage_info_.file_indexer_, user_comparator(),
                internal_comparator());
  // GetNextFile以二分方式获取下一个文件（L0全部参与）
  FdWithKeyRange* f = fp.GetNextFile();
  // 遍历查询
  while (f != nullptr) {
...
// 从缓存和SST文件内，sst文件的打开和关闭封装在table_cache_
    *status = table_cache_->Get(
        read_options, *internal_comparator(), *f->file_metadata, ikey,
        &get_context, mutable_cf_options_.block_protection_bytes_per_key,
        mutable_cf_options_.prefix_extractor,
        cfd_->internal_stats()->GetFileReadHist(fp.GetHitFileLevel()),
        IsFilterSkipped(static_cast<int>(fp.GetHitFileLevel()),
                        fp.IsHitFileLastInLevel()),
        fp.GetHitFileLevel(), max_file_size_for_l0_meta_pin_);
    ...
    f = fp.GetNextFile();
  }
  ...
}
```

