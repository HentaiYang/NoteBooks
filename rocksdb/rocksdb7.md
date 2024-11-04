当前笔记参考的作者是dgpp_programer，主要是他的[写性能最强的kv数据库RocksDB全集详解](https://www.bilibili.com/video/BV1vDWseEEys/?spm_id_from=333.999.0.0)系列

本文章同步到我的笔记：[https://github.com/HentaiYang/NoteBooks](https://github.com/HentaiYang/NoteBooks)

## 目录
* [1.后台Compaction顶层流程](#p1)
* &nbsp;&nbsp;[1.1.BGWorkCompaction & BackgroundCallCompaction](#p11)
* &nbsp;&nbsp;[1.2.BackgroundCompaction](#p12)
* &nbsp;&nbsp;&nbsp;&nbsp;[1.2.1.PickCompaction](#p121)
* &nbsp;&nbsp;&nbsp;&nbsp;[1.2.2.NeedsCompaction](#p122)
* [2.Compaction Job](#p2)
* &nbsp;&nbsp;[2.1.Run](#p21)
* &nbsp;&nbsp;[2.2.ProcessKeyValueCompaction](#p22)
* &nbsp;&nbsp;&nbsp;&nbsp;[2.2.1.MakeInputIterator](#p221)
* &nbsp;&nbsp;&nbsp;&nbsp;[2.2.2.NewCompactionMergingIterator](#p222)

---

# 1.后台Compaction顶层流程<a id="p1"></a>
## 1.1.BGWorkCompaction & BackgroundCallCompaction<a id="p11"></a>
本文介绍Compaction的执行流程，上篇文章提到自动和手动的compaction都会进入DBImpl::BGWorkCompaction入口，进行compaction流程。

和Flush类似，Compaction顶层调用顺序为**BGWorkCompaction - BackgroundCallCompaction - BackgroundCompaction**（Flush的对应调用只是把Compaction改成Flush）。

BGWorkCompaction会将自动触发的参数和prepicked_compaction实例化，而手动触发的则直接传参到BackgroundCallCompaction。
```cpp
void DBImpl::BGWorkCompaction(void* arg) {
  CompactionArg ca = *(static_cast<CompactionArg*>(arg));
  // 调用BGWorkCompation前会new一个CompactionArg，由本函数释放内存
  delete static_cast<CompactionArg*>(arg);
  IOSTATS_SET_THREAD_POOL_ID(Env::Priority::LOW);
  TEST_SYNC_POINT("DBImpl::BGWorkCompaction");
  // 手动触发的compaction，prepicked_compaction已经实例化，自动触发的还没实例化
  auto prepicked_compaction =
      static_cast<PrepickedCompaction*>(ca.prepicked_compaction);
  static_cast_with_check<DBImpl>(ca.db)->BackgroundCallCompaction(
      prepicked_compaction, Env::Priority::LOW);
  delete prepicked_compaction;
}
```

BackgroundCallCompaction同样会记录参与操作的最后一个文件号，执行BackgroundCompaction，并在执行后移除该文件号，并再次尝试调度MaybeScheduleFlushOrCompaction。

```cpp
void DBImpl::BackgroundCallCompaction(PrepickedCompaction* prepicked_compaction,
                                      Env::Priority bg_thread_pri) {
  bool made_progress = false;
  JobContext job_context(next_job_id_.fetch_add(1), true);
  TEST_SYNC_POINT("BackgroundCallCompaction:0");
  LogBuffer log_buffer(InfoLogLevel::INFO_LEVEL,
                       immutable_db_options_.info_log.get());
  {
    InstrumentedMutexLock l(&mutex_);

	num_running_compactions_++;
	// 记录参与compaction的最后一个文件号，完成后移除
    std::unique_ptr<std::list<uint64_t>::iterator>
        pending_outputs_inserted_elem(new std::list<uint64_t>::iterator(
            CaptureCurrentFileNumberInPendingOutputs()));

    assert((bg_thread_pri == Env::Priority::BOTTOM &&
            bg_bottom_compaction_scheduled_) ||
           (bg_thread_pri == Env::Priority::LOW && bg_compaction_scheduled_));
    // 进入后台compaction
    Status s = BackgroundCompaction(&made_progress, &job_context, &log_buffer,
                                    prepicked_compaction, bg_thread_pri);
    TEST_SYNC_POINT("BackgroundCallCompaction:1");
	// 失败处理 ...
	// 移除刚记录的文件号
	ReleaseFileNumberFromPendingOutputs(pending_outputs_inserted_elem);

    // 如果compaction失败，扫描出需要删除的文件
    FindObsoleteFiles(&job_context, !s.ok() && !s.IsShutdownInProgress() &&
                                        !s.IsManualCompactionPaused() &&
                                        !s.IsColumnFamilyDropped() &&
                                        !s.IsBusy());
    if (job_context.HaveSomethingToClean() ||
        job_context.HaveSomethingToDelete() || !log_buffer.IsEmpty()) {
      mutex_.Unlock();
      log_buffer.FlushBufferToLog();
      if (job_context.HaveSomethingToDelete()) {
        // 执行删除操作
        PurgeObsoleteFiles(job_context);
        TEST_SYNC_POINT("DBImpl::BackgroundCallCompaction:PurgedObsoleteFiles");
      }
      job_context.Clean();
      mutex_.Lock();
	}
	...
    // See if there's more work to be done
	MaybeScheduleFlushOrCompaction();
	...
  }
}
```

## 1.2.BackgroundCompaction<a id="p12"></a>

BackgroundCompaction执行流程较长，但总体是在组装compaction任务，然后进行一些收尾处理。

比较关键的函数有：

	PickCompaction：用于选取优先级最高的compaction。
	NeedsCompaction：用于确定选取一个compaction之后，是否需要继续compaction
	Run：正式运行Compaction job

```cpp
// 总体是在组装compaction任务
Status DBImpl::BackgroundCompaction(bool* made_progress,
                                    JobContext* job_context,
                                    LogBuffer* log_buffer,
                                    PrepickedCompaction* prepicked_compaction,
                                    Env::Priority thread_pri) {
  // 根据prepicked_compaction判断是否为手动触发
  ManualCompactionState* manual_compaction =
      prepicked_compaction == nullptr
          ? nullptr
          : prepicked_compaction->manual_compaction_state;
  *made_progress = false;
  mutex_.AssertHeld();

  TEST_SYNC_POINT("DBImpl::BackgroundCompaction:Start");
  const ReadOptions read_options(Env::IOActivity::kCompaction);
  const WriteOptions write_options(Env::IOActivity::kCompaction);
  
  bool is_manual = (manual_compaction != nullptr);
  std::unique_ptr<Compaction> c;
  if (prepicked_compaction != nullptr &&
      prepicked_compaction->compaction != nullptr) {
    // 手动触发，则直接将组装好的compaction交给c
    c.reset(prepicked_compaction->compaction);
  }
  bool is_prepicked = is_manual || c;

  // (manual_compaction->in_progress == false);
  // 是否禁用move机制（从一个level直接将sst移动到下一个level，不产生新sst）
  bool trivial_move_disallowed =
      is_manual && manual_compaction->disallow_trivial_move;

  CompactionJobStats compaction_job_stats;
  Status status;
  ...
  if (is_manual) {
    // another thread cannot pick up the same work
    manual_compaction->in_progress = true;
  }

  std::unique_ptr<TaskLimiterToken> task_token;

  bool sfm_reserved_compact_space = false;
  if (is_manual) {
    ManualCompactionState* m = manual_compaction;
    if (!c) {
      m->done = true;
      m->manual_end = nullptr;
      ...
    } else {
      // 检查有没有空间进行compaction
      bool enough_room = EnoughRoomForCompaction(
          m->cfd, *(c->inputs()), &sfm_reserved_compact_space, log_buffer);

      if (!enough_room) {
        // Then don't do the compaction
        c->ReleaseCompactionFiles(status);
        c.reset();
        status = Status::CompactionTooLarge();
      } else {
        ...打日志
      }
    }
  } else if (!is_prepicked && !compaction_queue_.empty()) {
    // 自动的
    if (HasExclusiveManualCompaction()) {
      // 自动compaction，但有手动compaction开启exclusive
      // 因此自动的就暂停，稍后执行
      unscheduled_compactions_++;
      return Status::OK();
	}
	// 从compaction_queue_队列取一个需要compaction的列族
    auto cfd = PickCompactionFromQueue(&task_token, log_buffer);
    if (cfd == nullptr) {
      ++unscheduled_compactions_;
      return Status::Busy();
	}
    // 通过引用计算判断这个列族是否还在使用，没有使用就不进行compaction
    if (cfd->UnrefAndTryDelete()) {
      return Status::OK();
    }
	// 获取当前列族最新的配置，位于mutable的配置都是可以在线修改的
	// 所以每次都去获取该列族最新的配置
    auto* mutable_cf_options = cfd->GetLatestMutableCFOptions();
    if (!mutable_cf_options->disable_auto_compactions && !cfd->IsDropped()) {
      // 从列族生成一个compaction然后交给c
      c.reset(cfd->PickCompaction(*mutable_cf_options, mutable_db_options_,
                                  log_buffer));
      if (c != nullptr) {
        bool enough_room = EnoughRoomForCompaction(
            cfd, *(c->inputs()), &sfm_reserved_compact_space, log_buffer);
        // 检查空间是否充足
        if (!enough_room) {
          // Then don't do the compaction
          ...
          // 将cfd放回队列
          AddToCompactionQueue(cfd);
          ++unscheduled_compactions_;
          c.reset();
          status = Status::CompactionTooLarge();
        } else {
          // update statistics
          size_t num_files = 0;
          for (auto& each_level : *c->inputs()) {
            num_files += each_level.files.size();
          }
          RecordInHistogram(stats_, NUM_FILES_IN_SINGLE_COMPACTION, num_files);
          // 从选取的列族中，判断是否还有新的需要执行compaction
          // 如果需要，就尝试触发一个新的调度来执行
          if (cfd->NeedsCompaction()) {
            // Yes, we need more compactions!
            AddToCompactionQueue(cfd);
            ++unscheduled_compactions_;
            MaybeScheduleFlushOrCompaction();
          }
        }
      }
    }
  }
  IOStatus io_s;
  bool compaction_released = false;
  if (!c) {
    // Nothing to do
    ROCKS_LOG_BUFFER(log_buffer, "Compaction nothing to do");
  } else if (c->deletion_compaction()) {
    // 如果compaction仅仅是删除文件，就可以直接删除（FIFO模式下会使用，Level模式没用）
	...
	// 从version中删除sst文件
    for (const auto& f : *c->inputs(0)) {
      c->edit()->DeleteFile(c->level(), f->fd.GetNumber());
	}
	// 落盘version到manifest文件
    status = versions_->LogAndApply(
        c->column_family_data(), *c->mutable_cf_options(), read_options,
        write_options, c->edit(), &mutex_, directories_.GetDbDir(),
        /*new_descriptor_log=*/false, /*column_family_options=*/nullptr,
        [&c, &compaction_released](const Status& s) {
          c->ReleaseCompactionFiles(s);
          compaction_released = true;
        });
	io_s = versions_->io_status();
	// 更新Superversion
    InstallSuperVersionAndScheduleWork(c->column_family_data(),
                                       &job_context->superversion_contexts[0],
                                       *c->mutable_cf_options());
    ...
  } else if (!trivial_move_disallowed && c->IsTrivialMove()) {
    // 如果input的文件和下一层没有重叠，则可以直接将sst move到下一层
    ThreadStatusUtil::SetColumnFamily(c->column_family_data());
	ThreadStatusUtil::SetThreadOperation(ThreadStatus::OP_COMPACTION);

	compaction_job_stats.num_input_files = c->num_input_files(0);

    NotifyOnCompactionBegin(c->column_family_data(), c.get(), status,
                            compaction_job_stats, job_context->job_id);

    // Move files to next level
    int32_t moved_files = 0;
    int64_t moved_bytes = 0;
    for (unsigned int l = 0; l < c->num_input_levels(); l++) {
      if (c->level(l) == c->output_level()) {
        continue;
      }
      for (size_t i = 0; i < c->num_input_files(l); i++) {
        FileMetaData* f = c->input(l, i);
        c->edit()->DeleteFile(c->level(l), f->fd.GetNumber());
        c->edit()->AddFile(一大堆参数);
        ++moved_files;
        moved_bytes += f->fd.GetFileSize();
      }
	}
	// 如果是因为当前层超过最大大小，则进行处理
    if (c->compaction_reason() == CompactionReason::kLevelMaxLevelSize &&
        c->immutable_options()->compaction_pri == kRoundRobin) {
      ...
	}
	// 更新version
    status = versions_->LogAndApply(
        c->column_family_data(), *c->mutable_cf_options(), read_options,
        write_options, c->edit(), &mutex_, directories_.GetDbDir(),
        /*new_descriptor_log=*/false, /*column_family_options=*/nullptr,
        [&c, &compaction_released](const Status& s) {
          c->ReleaseCompactionFiles(s);
          compaction_released = true;
        });
    io_s = versions_->io_status();
    // Use latest MutableCFOptions
    InstallSuperVersionAndScheduleWork(c->column_family_data(),
                                       &job_context->superversion_contexts[0],
                                       *c->mutable_cf_options());
    ...
  } else if (!is_prepicked && c->output_level() > 0 &&
             c->output_level() ==
                 c->column_family_data()
                     ->current()
                     ->storage_info()
                     ->MaxOutputLevel(
                         immutable_db_options_.allow_ingest_behind) &&
             env_->GetBackgroundThreads(Env::Priority::BOTTOM) > 0) {
    // 这里是最底层bottom compaction（level compaction一般不需要）
    ...
    env_->Schedule(&DBImpl::BGWorkBottomCompaction, ca, Env::Priority::BOTTOM,
                   this, &DBImpl::UnscheduleCompactionCallback);
  } else {
    // compaction进入最频繁的地方，在这里创建compaction_job
	...
	// 先创建一个compaction job
    CompactionJob compaction_job(
        job_context->job_id, c.get(), immutable_db_options_,
        mutable_db_options_, file_options_for_compaction_, versions_.get(),
        &shutting_down_, log_buffer, directories_.GetDbDir(),
        GetDataDir(c->column_family_data(), c->output_path_id()),
        GetDataDir(c->column_family_data(), 0), stats_, &mutex_,
        &error_handler_, snapshot_seqs, earliest_write_conflict_snapshot,
        snapshot_checker, job_context, table_cache_, &event_logger_,
        c->mutable_cf_options()->paranoid_file_checks,
        c->mutable_cf_options()->report_bg_io_stats, dbname_,
        &compaction_job_stats, thread_pri, io_tracer_,
        is_manual ? manual_compaction->canceled
                  : kManualCompactionCanceledFalse_,
        db_id_, db_session_id_, c->column_family_data()->GetFullHistoryTsLow(),
        c->trim_ts(), &blob_callback_, &bg_compaction_scheduled_,
        &bg_bottom_compaction_scheduled_);
    // 主要判断是否需要将compaction分为多个sub_compaction来并发执行，默认关闭
	compaction_job.Prepare();

    NotifyOnCompactionBegin(c->column_family_data(), c.get(), status,
                            compaction_job_stats, job_context->job_id);
    mutex_.Unlock();
    TEST_SYNC_POINT_CALLBACK(
        "DBImpl::BackgroundCompaction:NonTrivial:BeforeRun", nullptr);
    // 正式执行compaction
    compaction_job.Run().PermitUncheckedError();
    TEST_SYNC_POINT("DBImpl::BackgroundCompaction:NonTrivial:AfterRun");
	mutex_.Lock();
	// 固化compaction的执行结果，更新统计信息
	// 包括把input的sst文件从version删除，然后新生成sst文件加入到version
    status =
        compaction_job.Install(*c->mutable_cf_options(), &compaction_released);
    io_s = compaction_job.io_status();
	if (status.ok()) {
  	  // 固化superversion
      InstallSuperVersionAndScheduleWork(c->column_family_data(),
                                         &job_context->superversion_contexts[0],
                                         *c->mutable_cf_options());
    }
    *made_progress = true;
    TEST_SYNC_POINT_CALLBACK("DBImpl::BackgroundCompaction:AfterCompaction",
                             c->column_family_data());
  }

  if (status.ok() && !io_s.ok()) {
    status = io_s;
  } else {
    io_s.PermitUncheckedError();
  }

  if (c != nullptr) {
	if (!compaction_released) {
	  // 释放当前文件
      c->ReleaseCompactionFiles(status);
    } else {
#ifndef NDEBUG
      ...
#endif
	}
	*made_progress = true;

    // Need to make sure SstFileManager does its bookkeeping
    auto sfm = static_cast<SstFileManagerImpl*>(
        immutable_db_options_.sst_file_manager.get());
    if (sfm && sfm_reserved_compact_space) {
      sfm->OnCompactionCompletion(c.get());
	}
    NotifyOnCompactionCompleted(c->column_family_data(), c.get(), status,
                                compaction_job_stats, job_context->job_id);
  }
  ...

  if (is_manual) {
    ...
  }
  TEST_SYNC_POINT("DBImpl::BackgroundCompaction:Finish");
  return status;
}
```

---

### 1.2.1.PickCompaction<a id="p12"></a>

PickCompaction用于选取一个compaction，通过compaction_picker_生成。

```cpp
Compaction* ColumnFamilyData::PickCompaction(
    const MutableCFOptions& mutable_options,
const MutableDBOptions& mutable_db_options, LogBuffer* log_buffer) {
  // 通过compaction_picker_生成一个compaction
  auto* result = compaction_picker_->PickCompaction(
      GetName(), mutable_options, mutable_db_options, current_->storage_info(),
      log_buffer);
  if (result != nullptr) {
    result->FinalizeInputInfo(current_);
  }
  return result;
}
```

compaction_picker_通过builder生成compaction：

```cpp
Compaction* LevelCompactionPicker::PickCompaction(
    const std::string& cf_name, const MutableCFOptions& mutable_cf_options,
    const MutableDBOptions& mutable_db_options, VersionStorageInfo* vstorage,
    LogBuffer* log_buffer) {
  LevelCompactionBuilder builder(cf_name, vstorage, this, log_buffer,
                                 mutable_cf_options, ioptions_,
                                 mutable_db_options);
  return builder.PickCompaction();
}
```

builder优先选择score最大的compaction任务，然后在目标层级选一个sst文件，并扩展边界

```cpp
Compaction* LevelCompactionBuilder::PickCompaction() {
  // 根据compaction_score_和files_by_compaction_pri_选取sst文件
  // 先选score最大的，然后在目标层级选一个sst文件，然后扩展边界（CleanCut）
  SetupInitialFiles();
  if (start_level_inputs_.empty()) {
    return nullptr;
  }
  assert(start_level_ >= 0 && output_level_ >= 0);

  // 如果是L0，需要把L0的所有sst文件带上
  if (!SetupOtherL0FilesIfNeeded()) {
    return nullptr;
  }

  // 在output_level选取和input重合的sst文件，需要一起compaction，同时扩展边界
  if (!SetupOtherInputsIfNeeded()) {
    return nullptr;
  }

  // 根据上面的一些结果生成compaction，同时注册到compactions_in_progress_，更新score
  Compaction* c = GetCompaction();

  TEST_SYNC_POINT_CALLBACK("LevelCompactionPicker::PickCompaction:Return", c);

  return c;
}
```

---

### 1.2.2.NeedsCompaction<a id="p122"></a>

NeedsCompaction根据多个条件判断是否还需要进行compaction：

```cpp
// 选取一个compaction后，是否需要继续compaction
bool ColumnFamilyData::NeedsCompaction() const {
  return !mutable_cf_options_.disable_auto_compactions &&
         compaction_picker_->NeedsCompaction(current_->storage_info());
}

bool LevelCompactionPicker::NeedsCompaction(
const VersionStorageInfo* vstorage) const {‘’
  // 是否有过期sst文件（过期sst文件放在expired_ttl_files_）
  if (!vstorage->ExpiredTtlFiles().empty()) {
    return true;
  }
  // 是否有定期执行compaction的文件
  if (!vstorage->FilesMarkedForPeriodicCompaction().empty()) {
    return true;
  }
  // 是否有最底层sst被标记做compaction
  if (!vstorage->BottommostFilesMarkedForCompaction().empty()) {
    return true;
  }
  // 是否有sst被标记做compaction
  if (!vstorage->FilesMarkedForCompaction().empty()) {
    return true;
  }
  // 是否有sst被标记强制compaction
  if (!vstorage->FilesMarkedForForcedBlobGC().empty()) {
    return true;
  }
  // 遍历每一层，是否还有score大于1的level
  for (int i = 0; i <= vstorage->MaxInputLevel(); i++) {
    if (vstorage->CompactionScore(i) >= 1) {
      return true;
    }
  }
  return false;
}
```

---
# 2.Compaction Job<a id="p2"></a>
## 2.1.Run<a id="p21"></a>

Run方法会为所有sub_compaction（如果启用该功能则会有多个子compaction）分配线程执行ProcessKeyValueCompaction，并在本线程中对下表为0的任务执行ProcessKeyValueCompaction。

```cpp
Status CompactionJob::Run() {
  AutoThreadOperationStageUpdater stage_updater(
      ThreadStatus::STAGE_COMPACTION_RUN);
  TEST_SYNC_POINT("CompactionJob::Run():Start");
  log_buffer_->FlushBufferToLog();
  LogCompaction();

  const size_t num_threads = compact_->sub_compact_states.size();
  assert(num_threads > 0);
  const uint64_t start_micros = db_options_.clock->NowMicros();

  // 为前面分出的多个sub_compaction依次分配一个线程执行ProcessKeyValueCompaction
  std::vector<port::Thread> thread_pool;
  thread_pool.reserve(num_threads - 1);
  for (size_t i = 1; i < compact_->sub_compact_states.size(); i++) {
    thread_pool.emplace_back(&CompactionJob::ProcessKeyValueCompaction, this,
                             &compact_->sub_compact_states[i]);
  }

  // 合并Key value，这里面是真正合并的过程
  ProcessKeyValueCompaction(&compact_->sub_compact_states[0]);

  // 等待所有线程执行完成
  for (auto& thread : thread_pool) {
    thread.join();
  }
  // 更新统计信息和校验sst文件
  ...
  return status;
}
```

---

## 2.2.ProcessKeyValueCompaction<a id="p22"></a>

在这一步正式开始执行compaction，首先会判断db_options_.compaction_service，我的理解是这里判断是否使用外部Compaction算法，因为我的加速项目就是更换了这里面的ProcessKeyValueCompactionWithCompactionService函数为我们自己编写的Compaction方法。

然后是rocksdb的内部compaction算法，首先创建raw_input迭代器，以遍历input中sst的所有key value；创建MergeHelper，用来自定义Merge操作；创建c_iter，它的内部逻辑相当于小顶堆，根据内部的sst->Next()来确定堆顶，c_iter->Next()取用的就是堆顶的input的下一个key，然后将堆顶的Next()更新后，再根据新的所有input的Next()更新c_iter，使c_iter的堆顶一直是所有input中，input->Next()最小的那个input。

换句话说，sst内部key有序，因此sst->Next()肯定会取到当前sst内最小key，生成的新sst也需要key有序，因此就比较当前所有sst的首元素，将最小的那个插入，然后更新首元素->确认下一个最小Next()，有点像归并排序的其中一步，只不过这一步是考虑多个数组归并为一个数组。

插入到新sst文件用的方法是AddToOutput，完成新sst文件创建的方法是CloseCompactionFiles。

```cpp
void CompactionJob::ProcessKeyValueCompaction(SubcompactionState* sub_compact) {
  assert(sub_compact);
  assert(sub_compact->compaction);
  // 使用外部压缩算法
  if (db_options_.compaction_service) {
    CompactionServiceJobStatus comp_status =
        ProcessKeyValueCompactionWithCompactionService(sub_compact);
    if (comp_status == CompactionServiceJobStatus::kSuccess ||
        comp_status == CompactionServiceJobStatus::kFailure) {
      return;
    }
    // fallback to local compaction
    assert(comp_status == CompactionServiceJobStatus::kUseLocal);
  }

  uint64_t prev_cpu_micros = db_options_.clock->CPUMicros();

  ColumnFamilyData* cfd = sub_compact->compaction->column_family_data();

  // 配置了compaction_filter_factory时生成compaction_filter，默认没有
  const CompactionFilter* compaction_filter =
      cfd->ioptions()->compaction_filter;
  std::unique_ptr<CompactionFilter> compaction_filter_from_factory = nullptr;
  if (compaction_filter == nullptr) {
    compaction_filter_from_factory =
        sub_compact->compaction->CreateCompactionFilter();
    compaction_filter = compaction_filter_from_factory.get();
  }
  if (compaction_filter != nullptr && !compaction_filter->IgnoreSnapshots()) {
    sub_compact->status = Status::NotSupported(
        "CompactionFilter::IgnoreSnapshots() = false is not supported "
        "anymore.");
    return;
  }

  NotifyOnSubcompactionBegin(sub_compact);
  // 连续删除会放在range_del_agg里
  auto range_del_agg = std::make_unique<CompactionRangeDelAggregator>(
      &cfd->internal_comparator(), existing_snapshots_, &full_history_ts_low_,
      &trim_ts_);

  ...
  // 创建一个输入迭代器，这个迭代器能遍历所有input的sst里的key value
  std::unique_ptr<InternalIterator> raw_input(versions_->MakeInputIterator(
      read_options, sub_compact->compaction, range_del_agg.get(),
      file_options_for_read_, start, end));
  InternalIterator* input = raw_input.get();
  ...
  // 输入开始的位置
  if (start.has_value()) {
    start_ikey.SetInternalKey(*start, kMaxSequenceNumber, kValueTypeForSeek);
    if (ts_sz > 0) {
      start_ikey.UpdateInternalKey(kMaxSequenceNumber, kValueTypeForSeek,
                                   &ts_slice);
    }
    start_slice = start_ikey.GetInternalKey();
    start_user_key = start_ikey.GetUserKey();
  }
  // 输入结束的位置
  if (end.has_value()) {
    end_ikey.SetInternalKey(*end, kMaxSequenceNumber, kValueTypeForSeek);
    if (ts_sz > 0) {
      end_ikey.UpdateInternalKey(kMaxSequenceNumber, kValueTypeForSeek,
                                 &ts_slice);
    }
    end_slice = end_ikey.GetInternalKey();
    end_user_key = end_ikey.GetUserKey();
  }
  std::unique_ptr<InternalIterator> clip;
  // 更新input迭代器的开始、结束位置
  if (start.has_value() || end.has_value()) {
    clip = std::make_unique<ClippingIterator>(
        raw_input.get(), start.has_value() ? &start_slice : nullptr,
        end.has_value() ? &end_slice : nullptr, &cfd->internal_comparator());
    input = clip.get();
  }
  ...
  // 用于支持自定义合并操作
  MergeHelper merge(
      env_, cfd->user_comparator(), cfd->ioptions()->merge_operator.get(),
      compaction_filter, db_options_.info_log.get(),
      false /* internal key corruption is expected */,
      existing_snapshots_.empty() ? 0 : existing_snapshots_.back(),
      snapshot_checker_, compact_->compaction->level(), db_options_.stats);
  ...
  // CompactionIterator封装了compaction时的处理逻辑，包括4个组件：
  // InternalIterator* input_;  遍历输入的sst文件
  // MergeHelper* merge_helper_;  自定义的merge操作，默认不启用
  // std::vector<SequenceNumber>* snapshots_;  记录db现有快照，不在里面的合并后可丢弃
  // const CompactionFilter* compaction_filter_;  自定义的compaction操作，默认不启用
  // input_负责输入的key value，其余三个负责合并的处理逻辑
  auto c_iter = std::make_unique<CompactionIterator>(
      input, cfd->user_comparator(), &merge, versions_->LastSequence(),
      &existing_snapshots_, earliest_write_conflict_snapshot_, job_snapshot_seq,
      snapshot_checker_, env_, ShouldReportDetailedTime(env_, stats_),
      /*expect_valid_internal_key=*/true, range_del_agg.get(),
      blob_file_builder.get(), db_options_.allow_data_in_errors,
      db_options_.enforce_single_del_contracts, manual_compaction_canceled_,
      sub_compact->compaction
          ->DoesInputReferenceBlobFiles() /* must_count_input_entries */,
      sub_compact->compaction, compaction_filter, shutting_down_,
      db_options_.info_log, full_history_ts_low, preserve_time_min_seqno_,
      preclude_last_level_min_seqno_);
  // 封装了一些合并逻辑，以在移动迭代器时就做一些合并操作
  // 比如迭代器里面key的排序规则是：user_key正序，seqnum降序、valueType降序
  // 如果上一个key和当前key相同，valueType也相同，那么当前key的seq小于上个key
  // 因为seq是递增的，所以这个是旧key，没有快照的话就可以直接丢弃
  c_iter->SeekToFirst();

  // Assign range delete aggregator to the target output level, which makes sure
  // it only output to single level
  sub_compact->AssignRangeDelAggregator(std::move(range_del_agg));

  const auto& c_iter_stats = c_iter->iter_stats();

  // define the open and close functions for the compaction files, which will be
  // used open/close output files when needed.
  const CompactionFileOpenFunc open_file_func =
      [this, sub_compact](CompactionOutputs& outputs) {
        return this->OpenCompactionOutputFile(sub_compact, outputs);
      };
  const CompactionFileCloseFunc close_file_func =
      [this, sub_compact, start_user_key, end_user_key](
          CompactionOutputs& outputs, const Status& status,
          const Slice& next_table_min_key) {
        return this->FinishCompactionOutputFile(
            status, sub_compact, outputs, next_table_min_key,
            sub_compact->start.has_value() ? &start_user_key : nullptr,
            sub_compact->end.has_value() ? &end_user_key : nullptr);
      };

  Status status;
  TEST_SYNC_POINT_CALLBACK(
      "CompactionJob::ProcessKeyValueCompaction()::Processing",
      static_cast<void*>(const_cast<Compaction*>(sub_compact->compaction)));
  uint64_t last_cpu_micros = prev_cpu_micros;

  // 开始遍历key，然后写入到输出sst文件
  while (status.ok() && !cfd->IsDropped() && c_iter->Valid()) {
    // Invariant: c_iter.status() is guaranteed to be OK if c_iter->Valid()
    // returns true.
    assert(!end.has_value() ||
           cfd->user_comparator()->Compare(c_iter->user_key(), *end) < 0);

    if (c_iter_stats.num_input_records % kRecordStatsEvery ==
        kRecordStatsEvery - 1) {
      RecordDroppedKeys(c_iter_stats, &sub_compact->compaction_job_stats);
      c_iter->ResetRecordCounts();
      RecordCompactionIOStats();

      uint64_t cur_cpu_micros = db_options_.clock->CPUMicros();
      assert(cur_cpu_micros >= last_cpu_micros);
      RecordTick(stats_, COMPACTION_CPU_TOTAL_TIME,
                 cur_cpu_micros - last_cpu_micros);
      last_cpu_micros = cur_cpu_micros;
	}

    // Add current compaction_iterator key to target compaction output, if the
    // output file needs to be close or open, it will call the `open_file_func`
    // and `close_file_func`.
    // TODO: it would be better to have the compaction file open/close moved
    // into `CompactionOutputs` which has the output file information.
    status = sub_compact->AddToOutput(*c_iter, open_file_func, close_file_func);
    if (!status.ok()) {
      break;
	}

    c_iter->Next();
    if (c_iter->status().IsManualCompactionPaused()) {
      break;
    }
  }
  // 确认合并状态
  ...
  // 写入新的sst文件
  status = sub_compact->CloseCompactionFiles(status, open_file_func,
                                             close_file_func);
  if (blob_file_builder) {
    if (status.ok()) {
      status = blob_file_builder->Finish();
    } else {
      blob_file_builder->Abandon(status);
    }
    blob_file_builder.reset();
    sub_compact->Current().UpdateBlobStats();
  }
  // 更新统计信息，重置等
}
```

---
### 2.2.1.MakeInputIterator<a id="p221"></a>

MakeInputIterator用于创建sst的input迭代器：

```cpp
InternalIterator* VersionSet::MakeInputIterator(
    const ReadOptions& read_options, const Compaction* c,
    RangeDelAggregator* range_del_agg,
    const FileOptions& file_options_compactions,
    const std::optional<const Slice>& start,
    const std::optional<const Slice>& end) {
  auto cfd = c->column_family_data();
  // 如果输入是L0，那么为每个sst创建一个迭代器，为L1创建一个迭代器
  // 如果不是L0那就输入Level和输出Level各一个迭代器
  const size_t space = (c->level() == 0 ? c->input_levels(0)->num_files +
                                              c->num_input_levels() - 1
                                        : c->num_input_levels());
  InternalIterator** list = new InternalIterator*[space];
  std::vector<
      std::pair<TruncatedRangeDelIterator*, TruncatedRangeDelIterator***>>
      range_tombstones;
  size_t num = 0;
  // 让上面new的list数组中的每个迭代器指向具体的位置
  for (size_t which = 0; which < c->num_input_levels(); which++) {
if (c->input_levels(which)->num_files != 0) {
  // L0为每个sst生成一个迭代器
      if (c->level(which) == 0) {
        const LevelFilesBrief* flevel = c->input_levels(which);
        for (size_t i = 0; i < flevel->num_files; i++) {
          const FileMetaData& fmd = *flevel->files[i].file_metadata;
          if (start.has_value() &&
              cfd->user_comparator()->CompareWithoutTimestamp(
                  *start, fmd.largest.user_key()) > 0) {
            continue;
          }
          // We should be able to filter out the case where the end key
          // equals to the end boundary, since the end key is exclusive.
          // We try to be extra safe here.
          if (end.has_value() &&
              cfd->user_comparator()->CompareWithoutTimestamp(
                  *end, fmd.smallest.user_key()) < 0) {
            continue;
          }
          TruncatedRangeDelIterator* range_tombstone_iter = nullptr;
          list[num++] = cfd->table_cache()->NewIterator(
              read_options, file_options_compactions,
              cfd->internal_comparator(), fmd, range_del_agg,
              c->mutable_cf_options()->prefix_extractor,
              /*table_reader_ptr=*/nullptr,
              /*file_read_hist=*/nullptr, TableReaderCaller::kCompaction,
              /*arena=*/nullptr,
              /*skip_filters=*/false,
              /*level=*/static_cast<int>(c->level(which)),
              MaxFileSizeForL0MetaPin(*c->mutable_cf_options()),
              /*smallest_compaction_key=*/nullptr,
              /*largest_compaction_key=*/nullptr,
              /*allow_unprepared_value=*/false,
              c->mutable_cf_options()->block_protection_bytes_per_key,
              /*range_del_read_seqno=*/nullptr,
              /*range_del_iter=*/&range_tombstone_iter);
          range_tombstones.emplace_back(range_tombstone_iter, nullptr);
        }
      } else {
        // 非L0就为每层level生成一个迭代器
        // Create concatenating iterator for the files from this level
        TruncatedRangeDelIterator*** tombstone_iter_ptr = nullptr;
        list[num++] = new LevelIterator(
            cfd->table_cache(), read_options, file_options_compactions,
            cfd->internal_comparator(), c->input_levels(which),
            c->mutable_cf_options()->prefix_extractor,
            /*should_sample=*/false,
            /*no per level latency histogram=*/nullptr,
            TableReaderCaller::kCompaction, /*skip_filters=*/false,
            /*level=*/static_cast<int>(c->level(which)),
            c->mutable_cf_options()->block_protection_bytes_per_key,
            range_del_agg, c->boundaries(which), false, &tombstone_iter_ptr);
        range_tombstones.emplace_back(nullptr, tombstone_iter_ptr);
      }
    }
  }
  assert(num <= space);
  // 根据迭代器集合list，生成MergeingIterator，从而遍历所有sst文件的kv
  InternalIterator* result = NewCompactionMergingIterator(
      &c->column_family_data()->internal_comparator(), list,
      static_cast<int>(num), range_tombstones);
  delete[] list;
  return result;
}
```

### 2.2.2.NewCompactionMergingIterator<a id="p222"></a>

用于创建c_iter，创建一个小顶堆：

```cpp
InternalIterator* NewCompactionMergingIterator(
    const InternalKeyComparator* comparator, InternalIterator** children, int n,
    std::vector<std::pair<TruncatedRangeDelIterator*,
                          TruncatedRangeDelIterator***>>& range_tombstone_iters,
    Arena* arena) {
  assert(n >= 0);
  if (n == 0) {
    return NewEmptyInternalIterator<Slice>(arena);
  } else {
    if (arena == nullptr) {
      return new CompactionMergingIterator(comparator, children, n,
                                           false /* is_arena_mode */,
                                           range_tombstone_iters);
    } else {
      auto mem = arena->AllocateAligned(sizeof(CompactionMergingIterator));
      return new (mem) CompactionMergingIterator(comparator, children, n,
                                                 true /* is_arena_mode */,
                                                 range_tombstone_iters);
    }
  }
}
```

该类的初始化流程如下：

```cpp
// 管理多个子迭代器形成一个最小堆，Next调用堆顶迭代器的Next
// 然后根据该迭代器新的最小key，生成新的堆顶
CompactionMergingIterator(
    const InternalKeyComparator* comparator, InternalIterator** children,
    int n, bool is_arena_mode,
    std::vector<
        std::pair<TruncatedRangeDelIterator*, TruncatedRangeDelIterator***>>
        range_tombstones)
    : is_arena_mode_(is_arena_mode),
      comparator_(comparator),
      current_(nullptr),
      minHeap_(CompactionHeapItemComparator(comparator_)),
      pinned_iters_mgr_(nullptr) {
  children_.resize(n);
  for (int i = 0; i < n; i++) {
    children_[i].level = i;
    children_[i].iter.Set(children[i]);
    assert(children_[i].type == HeapItem::ITERATOR);
  }
  assert(range_tombstones.size() == static_cast<size_t>(n));
  for (auto& p : range_tombstones) {
    range_tombstone_iters_.push_back(p.first);
  }
  pinned_heap_item_.resize(n);
  for (int i = 0; i < n; ++i) {
    if (range_tombstones[i].second) {
      // for LevelIterator
      *range_tombstones[i].second = &range_tombstone_iters_[i];
    }
    pinned_heap_item_[i].level = i;
    pinned_heap_item_[i].type = HeapItem::DELETE_RANGE_START;
  }
}
```

---

至此，rocksdb的compaction流程也分析完毕，对rocksdb的学习暂时告一段落，勉强算是入了点门吧，后续会从基础的ceph开始学习分布式，然后对目前项目而言还需要继续学习FPGA，所以后面应该会继续更新FPGA的笔记。