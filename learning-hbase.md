# HBase原理和源码 #
2019/9/18 23:19:40  

## 参考资料 ##
- [有态度的HBase/Spark/BigData - HBase](http://hbasefly.com/?vilqlm=bnem43&xgrony=vo0822)  
- [Openinx Blog](http://openinx.github.io/)
- [岑玉海 - hbase](https://cloud.tencent.com/developer/column/1908/tag-10824)
- 《HBase不睡觉书》

## 存储架构 ##

![](https://i.imgur.com/CUXyzuj.png)

## 存储模型 ##
![](https://i.imgur.com/D7khY0x.png)

## 写流程 ##

- [HBase － 数据写入流程解析](http://hbasefly.com/2016/03/23/hbase_writer/)：HBase-1.x版本的写流程中，某次补丁对"写HLog"的优化
- [HBase最佳实践－写性能优化策略](http://hbasefly.com/2016/12/10/hbase-parctice-write/)
- [hbase源码系列（十一）Put、Delete在服务端是如何处理？](https://cloud.tencent.com/developer/article/1048107)

![](https://i.imgur.com/dTfHM1P.png)  
1）Client 先访问 zookeeper，获取 hbase:meta 表位于哪个 Region Server。  
2）访问对应的 Region Server，获取 hbase:meta 表，根据读请求的 namespace:table/rowkey，查询出目标数据位于哪个 Region Server 中的哪个 Region 中。并将该 table 的 region 信息以及 meta 表的位置信息缓存在客户端的 meta cache，方便下次访问。  
3）与目标 Region Server 进行通讯；  
4）将数据顺序写入（追加）到 WAL；  
5）将数据写入对应的 MemStore，数据会在 MemStore 进行排序；  
6）向客户端发送 ack；  
7）等达到 MemStore 的刷写时机后，将数据刷写到 HFile。  
8）HBase-1.x源码

 ```java
package org.apache.hadoop.hbase.regionserver.HRegion;

  @SuppressWarnings("unchecked")
  private long doMiniBatchMutation(BatchOperationInProgress<?> batchOp) throws IOException {
    boolean isInReplay = batchOp.isInReplay();
    // variable to note if all Put items are for the same CF -- metrics related
    boolean putsCfSetConsistent = true;
    //The set of columnFamilies first seen for Put.
    Set<byte[]> putsCfSet = null;
    // variable to note if all Delete items are for the same CF -- metrics related
    boolean deletesCfSetConsistent = true;
    //The set of columnFamilies first seen for Delete.
    Set<byte[]> deletesCfSet = null;

    long currentNonceGroup = HConstants.NO_NONCE, currentNonce = HConstants.NO_NONCE;
    WALEdit walEdit = null;
    // dolly: 多版本并发控制
    MultiVersionConcurrencyControl.WriteEntry writeEntry = null;
    long txid = 0;
    boolean doRollBackMemstore = false;
    boolean locked = false;
    int cellCount = 0;
    /** Keep track of the locks we hold so we can release them in finally clause */
    List<RowLock> acquiredRowLocks = Lists.newArrayListWithCapacity(batchOp.operations.length);
    // reference family maps directly so coprocessors can mutate them if desired
    Map<byte[], List<Cell>>[] familyMaps = new Map[batchOp.operations.length];
    // We try to set up a batch in the range [firstIndex,lastIndexExclusive)
    int firstIndex = batchOp.nextIndexToProcess;
    int lastIndexExclusive = firstIndex;
    boolean success = false;
    int noOfPuts = 0, noOfDeletes = 0;
    WALKey walKey = null;
    long mvccNum = 0;
    long addedSize = 0;
    try {
      // ------------------------------------
      // STEP 1. Try to acquire as many locks as we can, and ensure
      // we acquire at least one.
	  // dolly: 获取行锁和Region更新共享锁, 且保证读写分离; 下面这一大堆是需要考虑到的异常情况
      // ----------------------------------
      int numReadyToWrite = 0;
      long now = EnvironmentEdgeManager.currentTime();
      while (lastIndexExclusive < batchOp.operations.length) {
        Mutation mutation = batchOp.getMutation(lastIndexExclusive);
        boolean isPutMutation = mutation instanceof Put;

        Map<byte[], List<Cell>> familyMap = mutation.getFamilyCellMap();
        // store the family map reference to allow for mutations
        familyMaps[lastIndexExclusive] = familyMap;

        // skip anything that "ran" already
        if (batchOp.retCodeDetails[lastIndexExclusive].getOperationStatusCode()
            != OperationStatusCode.NOT_RUN) {
          lastIndexExclusive++;
          continue;
        }

        try {
          checkAndPrepareMutation(mutation, batchOp.isInReplay(), familyMap, now);
        } catch (NoSuchColumnFamilyException nscf) {
          LOG.warn("No such column family in batch mutation", nscf);
          batchOp.retCodeDetails[lastIndexExclusive] = new OperationStatus(
              OperationStatusCode.BAD_FAMILY, nscf.getMessage());
          lastIndexExclusive++;
          continue;
        } catch (FailedSanityCheckException fsce) {
          LOG.warn("Batch Mutation did not pass sanity check", fsce);
          batchOp.retCodeDetails[lastIndexExclusive] = new OperationStatus(
              OperationStatusCode.SANITY_CHECK_FAILURE, fsce.getMessage());
          lastIndexExclusive++;
          continue;
        } catch (WrongRegionException we) {
          LOG.warn("Batch mutation had a row that does not belong to this region", we);
          batchOp.retCodeDetails[lastIndexExclusive] = new OperationStatus(
              OperationStatusCode.SANITY_CHECK_FAILURE, we.getMessage());
          lastIndexExclusive++;
          continue;
        }

        // If we haven't got any rows in our batch, we should block to
        // get the next one.
        RowLock rowLock = null;
        try {
          // dolly: 可以看到HBase只支持单行事务
          rowLock = getRowLockInternal(mutation.getRow(), true);
        } catch (IOException ioe) {
          LOG.warn("Failed getting lock in batch put, row="
            + Bytes.toStringBinary(mutation.getRow()), ioe);
        }
        if (rowLock == null) {
          // We failed to grab another lock
          break; // stop acquiring more rows for this batch
        } else {
          acquiredRowLocks.add(rowLock);
        }

        lastIndexExclusive++;
        numReadyToWrite++;
        if (isInReplay) {
          for (List<Cell> cells : mutation.getFamilyCellMap().values()) {
            cellCount += cells.size();
          }
        }
        if (isPutMutation) {
          // If Column Families stay consistent through out all of the
          // individual puts then metrics can be reported as a mutliput across
          // column families in the first put.
          if (putsCfSet == null) {
            putsCfSet = mutation.getFamilyCellMap().keySet();
          } else {
            putsCfSetConsistent = putsCfSetConsistent
                && mutation.getFamilyCellMap().keySet().equals(putsCfSet);
          }
        } else {
          if (deletesCfSet == null) {
            deletesCfSet = mutation.getFamilyCellMap().keySet();
          } else {
            deletesCfSetConsistent = deletesCfSetConsistent
                && mutation.getFamilyCellMap().keySet().equals(deletesCfSet);
          }
        }
      }

      // we should record the timestamp only after we have acquired the rowLock,
      // otherwise, newer puts/deletes are not guaranteed to have a newer timestamp
      // dolly: 该时间戳为服务端的时间
      now = EnvironmentEdgeManager.currentTime();
      byte[] byteNow = Bytes.toBytes(now);

      // Nothing to put/delete -- an exception in the above such as NoSuchColumnFamily?
      if (numReadyToWrite <= 0) return 0L;

      // We've now grabbed as many mutations off the list as we can

      // ------------------------------------
      // STEP 2. Update any LATEST_TIMESTAMP timestamps
      // dolly: 在Put类中, 若不指定时间戳, 会置为Long.MAX_VALUE, 在第二步中更新
      // ----------------------------------
      for (int i = firstIndex; !isInReplay && i < lastIndexExclusive; i++) {
        // skip invalid
        if (batchOp.retCodeDetails[i].getOperationStatusCode()
            != OperationStatusCode.NOT_RUN) continue;

        Mutation mutation = batchOp.getMutation(i);
        // dolly: Put和Delete两种操作会对时间戳有不同的处理方式, 其余相同
        if (mutation instanceof Put) {
          updateCellTimestamps(familyMaps[i].values(), byteNow);
          noOfPuts++;
        } else {
          prepareDeleteTimestamps(mutation, familyMaps[i], byteNow);
          noOfDeletes++;
        }
        rewriteCellTags(familyMaps[i], mutation);
        WALEdit fromCP = batchOp.walEditsFromCoprocessors[i];
        if (fromCP != null) {
          cellCount += fromCP.size();
        }
        for (List<Cell> cells : familyMaps[i].values()) {
          cellCount += cells.size();
        }
      }
      walEdit = new WALEdit(cellCount, isInReplay);
      // 读操作上锁
      lock(this.updatesLock.readLock(), numReadyToWrite);
      locked = true;

      // calling the pre CP hook for batch mutation
      if (!isInReplay && coprocessorHost != null) {
        MiniBatchOperationInProgress<Mutation> miniBatchOp =
          new MiniBatchOperationInProgress<Mutation>(batchOp.getMutationsForCoprocs(),
          batchOp.retCodeDetails, batchOp.walEditsFromCoprocessors, firstIndex, lastIndexExclusive);
        if (coprocessorHost.preBatchMutate(miniBatchOp)) {
          return 0L;
        } else {
          for (int i = firstIndex; i < lastIndexExclusive; i++) {
            if (batchOp.retCodeDetails[i].getOperationStatusCode() != OperationStatusCode.NOT_RUN) {
              // lastIndexExclusive was incremented above.
              continue;
            }
            // we pass (i - firstIndex) below since the call expects a relative index
            Mutation[] cpMutations = miniBatchOp.getOperationsFromCoprocessors(i - firstIndex);
            if (cpMutations == null) {
              continue;
            }
            // Else Coprocessor added more Mutations corresponding to the Mutation at this index.
            for (int j = 0; j < cpMutations.length; j++) {
              Mutation cpMutation = cpMutations[j];
              Map<byte[], List<Cell>> cpFamilyMap = cpMutation.getFamilyCellMap();
              checkAndPrepareMutation(cpMutation, isInReplay, cpFamilyMap, now);

              // Acquire row locks. If not, the whole batch will fail.
              acquiredRowLocks.add(getRowLockInternal(cpMutation.getRow(), true));

              if (cpMutation.getDurability() == Durability.SKIP_WAL) {
                recordMutationWithoutWal(cpFamilyMap);
              }

              // Returned mutations from coprocessor correspond to the Mutation at index i. We can
              // directly add the cells from those mutations to the familyMaps of this mutation.
              mergeFamilyMaps(familyMaps[i], cpFamilyMap); // will get added to the memstore later
            }
          }
        }
      }

      // ------------------------------------
      // STEP 3. Build WAL edit
	  // dolly: 在内存中构建WAL的修改对象
      // ----------------------------------
      Durability durability = Durability.USE_DEFAULT;
      for (int i = firstIndex; i < lastIndexExclusive; i++) {
        // Skip puts that were determined to be invalid during preprocessing
        if (batchOp.retCodeDetails[i].getOperationStatusCode() != OperationStatusCode.NOT_RUN) {
          continue;
        }

        Mutation m = batchOp.getMutation(i);
        Durability tmpDur = getEffectiveDurability(m.getDurability());
        if (tmpDur.ordinal() > durability.ordinal()) {
          durability = tmpDur;
        }
        if (tmpDur == Durability.SKIP_WAL) {
          recordMutationWithoutWal(m.getFamilyCellMap());
          continue;
        }

        long nonceGroup = batchOp.getNonceGroup(i), nonce = batchOp.getNonce(i);
        // In replay, the batch may contain multiple nonces. If so, write WALEdit for each.
        // Given how nonces are originally written, these should be contiguous.
        // They don't have to be, it will still work, just write more WALEdits than needed.
        if (nonceGroup != currentNonceGroup || nonce != currentNonce) {
          if (walEdit.size() > 0) {
            assert isInReplay;
            if (!isInReplay) {
              throw new IOException("Multiple nonces per batch and not in replay");
            }
            // txid should always increase, so having the one from the last call is ok.
            // we use HLogKey here instead of WALKey directly to support legacy coprocessors.
            walKey = new ReplayHLogKey(this.getRegionInfo().getEncodedNameAsBytes(),
              this.htableDescriptor.getTableName(), now, m.getClusterIds(),
              currentNonceGroup, currentNonce, mvcc);
            txid = this.wal.append(this.htableDescriptor,  this.getRegionInfo(),  walKey,
              walEdit, true);
            walEdit = new WALEdit(cellCount, isInReplay);
            walKey = null;
          }
          currentNonceGroup = nonceGroup;
          currentNonce = nonce;
        }

        // Add WAL edits by CP
        WALEdit fromCP = batchOp.walEditsFromCoprocessors[i];
        if (fromCP != null) {
          for (Cell cell : fromCP.getCells()) {
            walEdit.add(cell);
          }
        }
        addFamilyMapToWALEdit(familyMaps[i], walEdit);
      }

      // -------------------------
      // STEP 4. Append the final edit to WAL. Do not sync wal.
	  // dolly: 在内存中, 把WAL的修改对象(异步)追加到WAL之后, 但不写到HDFS
      // -------------------------
      Mutation mutation = batchOp.getMutation(firstIndex);
      if (isInReplay) {
        // use wal key from the original
        walKey = new ReplayHLogKey(this.getRegionInfo().getEncodedNameAsBytes(),
          this.htableDescriptor.getTableName(), WALKey.NO_SEQUENCE_ID, now,
          mutation.getClusterIds(), currentNonceGroup, currentNonce, mvcc);
        long replaySeqId = batchOp.getReplaySequenceId();
        walKey.setOrigLogSeqNum(replaySeqId);
      }
      if (walEdit.size() > 0) {
        if (!isInReplay) {
        // we use HLogKey here instead of WALKey directly to support legacy coprocessors.
        walKey = new HLogKey(this.getRegionInfo().getEncodedNameAsBytes(),
            this.htableDescriptor.getTableName(), WALKey.NO_SEQUENCE_ID, now,
            mutation.getClusterIds(), currentNonceGroup, currentNonce, mvcc);
        }
        txid = this.wal.append(this.htableDescriptor, this.getRegionInfo(), walKey, walEdit, true);
      }
      // ------------------------------------
      // Acquire the latest mvcc number
      // ----------------------------------
      if (walKey == null) {
        // If this is a skip wal operation just get the read point from mvcc
        walKey = this.appendEmptyEdit(this.wal);
      }
      if (!isInReplay) {
        writeEntry = walKey.getWriteEntry();
        mvccNum = writeEntry.getWriteNumber();
      } else {
        mvccNum = batchOp.getReplaySequenceId();
      }

      // ------------------------------------
      // STEP 5. Write back to memstore
      // Write to memstore. It is ok to write to memstore
      // first without syncing the WAL because we do not roll
      // forward the memstore MVCC. The MVCC will be moved up when
      // the complete operation is done. These changes are not yet
      // visible to scanners till we update the MVCC. The MVCC is
      // moved only when the sync is complete.
      // ----------------------------------
      for (int i = firstIndex; i < lastIndexExclusive; i++) {
        if (batchOp.retCodeDetails[i].getOperationStatusCode()
            != OperationStatusCode.NOT_RUN) {
          continue;
        }
        doRollBackMemstore = true; // If we have a failure, we need to clean what we wrote
        addedSize += applyFamilyMapToMemstore(familyMaps[i], mvccNum, isInReplay);
      }

      // -------------------------------
      // STEP 6. Release row locks, etc.
	  // dolly: 释放锁, 但此时仍然不能读到刚刚写入的数据
      // -------------------------------
      if (locked) {
        this.updatesLock.readLock().unlock();
        locked = false;
      }
      releaseRowLocks(acquiredRowLocks);

      // -------------------------
      // STEP 7. Sync wal.
	  // dolly: 将WAL写到HDFS
      // -------------------------
      if (txid != 0) {
        syncOrDefer(txid, durability);
      }

	  // dolly: 此时WAL和MemStore都在内存中, 如果RegionServer挂了则回滚
      doRollBackMemstore = false;
      // calling the post CP hook for batch mutation
      if (!isInReplay && coprocessorHost != null) {
        MiniBatchOperationInProgress<Mutation> miniBatchOp =
          new MiniBatchOperationInProgress<Mutation>(batchOp.getMutationsForCoprocs(),
          batchOp.retCodeDetails, batchOp.walEditsFromCoprocessors, firstIndex, lastIndexExclusive);
        coprocessorHost.postBatchMutate(miniBatchOp);
      }

      // ------------------------------------------------------------------
      // STEP 8. Advance mvcc. This will make this put visible to scanners and getters.
	  // dolly: 更新MultiVersionConcurrencyControl对象, 此时可以读到数据
      // ------------------------------------------------------------------
      if (writeEntry != null) {
        mvcc.completeAndWait(writeEntry);
        writeEntry = null;
      } else if (isInReplay) {
        // ensure that the sequence id of the region is at least as big as orig log seq id
        mvcc.advanceTo(mvccNum);
      }

      for (int i = firstIndex; i < lastIndexExclusive; i ++) {
        if (batchOp.retCodeDetails[i] == OperationStatus.NOT_RUN) {
          batchOp.retCodeDetails[i] = OperationStatus.SUCCESS;
        }
      }

      // ------------------------------------
      // STEP 9. Run coprocessor post hooks. This should be done after the wal is
      // synced so that the coprocessor contract is adhered to.
	  // dolly: 执行协处理器相关操作, 必须在WAL已经同步后才会执行
      // ------------------------------------
      if (!isInReplay && coprocessorHost != null) {
        for (int i = firstIndex; i < lastIndexExclusive; i++) {
          // only for successful puts
          if (batchOp.retCodeDetails[i].getOperationStatusCode()
              != OperationStatusCode.SUCCESS) {
            continue;
          }
          Mutation m = batchOp.getMutation(i);
          if (m instanceof Put) {
            coprocessorHost.postPut((Put) m, walEdit, m.getDurability());
          } else {
            coprocessorHost.postDelete((Delete) m, walEdit, m.getDurability());
          }
        }
      }

      success = true;
      return addedSize;
    } finally {
      // if the wal sync was unsuccessful, remove keys from memstore
      if (doRollBackMemstore) {
        for (int j = 0; j < familyMaps.length; j++) {
          for(List<Cell> cells:familyMaps[j].values()) {
            rollbackMemstore(cells);
          }
        }
        if (writeEntry != null) mvcc.complete(writeEntry);
      } else {
        this.addAndGetGlobalMemstoreSize(addedSize);
        if (writeEntry != null) {
          mvcc.completeAndWait(writeEntry);
        }
      }

      if (locked) {
        this.updatesLock.readLock().unlock();
      }
      releaseRowLocks(acquiredRowLocks);

      // See if the column families were consistent through the whole thing.
      // if they were then keep them. If they were not then pass a null.
      // null will be treated as unknown.
      // Total time taken might be involving Puts and Deletes.
      // Split the time for puts and deletes based on the total number of Puts and Deletes.

      if (noOfPuts > 0) {
        // There were some Puts in the batch.
        if (this.metricsRegion != null) {
          this.metricsRegion.updatePut();
        }
      }
      if (noOfDeletes > 0) {
        // There were some Deletes in the batch.
        if (this.metricsRegion != null) {
          this.metricsRegion.updateDelete();
        }
      }
      if (!success) {
        for (int i = firstIndex; i < lastIndexExclusive; i++) {
          if (batchOp.retCodeDetails[i].getOperationStatusCode() == OperationStatusCode.NOT_RUN) {
            batchOp.retCodeDetails[i] = OperationStatus.FAILURE;
          }
        }
      }
      if (coprocessorHost != null && !batchOp.isInReplay()) {
        // call the coprocessor hook to do any finalization steps
        // after the put is done
        MiniBatchOperationInProgress<Mutation> miniBatchOp =
          new MiniBatchOperationInProgress<Mutation>(batchOp.getMutationsForCoprocs(),
          batchOp.retCodeDetails, batchOp.walEditsFromCoprocessors, firstIndex, lastIndexExclusive);
        coprocessorHost.postBatchMutateIndispensably(miniBatchOp, success);
      }

      batchOp.nextIndexToProcess = lastIndexExclusive;
    }
  }
 ```
9）HBase-2.x源码

 ```java
package org.apache.hadoop.hbase.regionserver.HRegion;  

  /**
   * Called to do a piece of the batch that came in to {@link #batchMutatpe(Mutation[], long, long)}
   * In here we also handle replay of edits on region recover. Also gets change in size brought
   * about by applying {@code batchOp}.
   */
  private void doMiniBatchMutate(BatchOperation<?> batchOp) throws IOException {
    boolean success = false;
    WALEdit walEdit = null;
    WriteEntry writeEntry = null;
    boolean locked = false;
    // We try to set up a batch in the range [batchOp.nextIndexToProcess,lastIndexExclusive)
    MiniBatchOperationInProgress<Mutation> miniBatchOp = null;
    /** Keep track of the locks we hold so we can release them in finally clause */
    List<RowLock> acquiredRowLocks = Lists.newArrayListWithCapacity(batchOp.size());
    try {
      // STEP 1. Try to acquire as many locks as we can and build mini-batch of operations with
      // locked rows
      miniBatchOp = batchOp.lockRowsAndBuildMiniBatch(acquiredRowLocks);

      // We've now grabbed as many mutations off the list as we can
      // Ensure we acquire at least one.
      if (miniBatchOp.getReadyToWriteCount() <= 0) {
        // Nothing to put/delete -- an exception in the above such as NoSuchColumnFamily?
        return;
      }

      lock(this.updatesLock.readLock(), miniBatchOp.getReadyToWriteCount());
      locked = true;

      // STEP 2. Update mini batch of all operations in progress with  LATEST_TIMESTAMP timestamp
      // We should record the timestamp only after we have acquired the rowLock,
      // otherwise, newer puts/deletes are not guaranteed to have a newer timestamp
      long now = EnvironmentEdgeManager.currentTime();
      batchOp.prepareMiniBatchOperations(miniBatchOp, now, acquiredRowLocks);

      // STEP 3. Build WAL edit
      List<Pair<NonceKey, WALEdit>> walEdits = batchOp.buildWALEdits(miniBatchOp);

      // STEP 4. Append the WALEdits to WAL and sync.
	  // dolly: 和1.x不同, 在内存中构建完WAL后会直接写到HDFS上
      for(Iterator<Pair<NonceKey, WALEdit>> it = walEdits.iterator(); it.hasNext();) {
        Pair<NonceKey, WALEdit> nonceKeyWALEditPair = it.next();
        walEdit = nonceKeyWALEditPair.getSecond();
        NonceKey nonceKey = nonceKeyWALEditPair.getFirst();

        if (walEdit != null && !walEdit.isEmpty()) {
          writeEntry = doWALAppend(walEdit, batchOp.durability, batchOp.getClusterIds(), now,
              nonceKey.getNonceGroup(), nonceKey.getNonce(), batchOp.getOrigLogSeqNum());
        }

        // Complete mvcc for all but last writeEntry (for replay case)
        if (it.hasNext() && writeEntry != null) {
          mvcc.complete(writeEntry);
          writeEntry = null;
        }
      }

      // STEP 5. Write back to memStore
      // NOTE: writeEntry can be null here
      writeEntry = batchOp.writeMiniBatchOperationsToMemStore(miniBatchOp, writeEntry);

      // STEP 6. Complete MiniBatchOperations: If required calls postBatchMutate() CP hook and
      // complete mvcc for last writeEntry
      batchOp.completeMiniBatchOperations(miniBatchOp, writeEntry);
      writeEntry = null;
      success = true;
    } finally {
      // Call complete rather than completeAndWait because we probably had error if walKey != null
      if (writeEntry != null) mvcc.complete(writeEntry);

      if (locked) {
        this.updatesLock.readLock().unlock();
      }
      releaseRowLocks(acquiredRowLocks);

      final int finalLastIndexExclusive =
          miniBatchOp != null ? miniBatchOp.getLastIndexExclusive() : batchOp.size();
      final boolean finalSuccess = success;
      batchOp.visitBatchOperations(true, finalLastIndexExclusive, (int i) -> {
        batchOp.retCodeDetails[i] =
            finalSuccess ? OperationStatus.SUCCESS : OperationStatus.FAILURE;
        return true;
      });

      batchOp.doPostOpCleanupForMiniBatch(miniBatchOp, walEdit, finalSuccess);

      batchOp.nextIndexToProcess = finalLastIndexExclusive;
    }
  }
 ```

## 读流程

- [HBase原理－数据读取流程解析](http://hbasefly.com/2016/12/21/hbase-getorscan/)
- [HBase原理－迟到的‘数据读取流程’部分细节](http://hbasefly.com/2017/06/11/hbase-scan-2/)
- [HBase最佳实践 – Scan用法大观园](http://hbasefly.com/2017/10/29/hbase-scan-3/)
- [HBase最佳实践－读性能优化策略](http://hbasefly.com/2016/11/11/hbase%e6%9c%80%e4%bd%b3%e5%ae%9e%e8%b7%b5%ef%bc%8d%e8%af%bb%e6%80%a7%e8%83%bd%e4%bc%98%e5%8c%96%e7%ad%96%e7%95%a5/)
- [hbase源码系列（十二）Get、Scan在服务端是如何处理？](https://cloud.tencent.com/developer/article/1048112)
- [hbase源码系列（十五）终结篇&Scan续集-->如何查询出来下一个KeyValue](https://cloud.tencent.com/developer/article/1047866)

![](https://i.imgur.com/rlOHKlj.png)  
1）Client 先访问 zookeeper，获取 hbase:meta 表位于哪个 Region Server。  
2）访问对应的 Region Server，获取 hbase:meta 表，根据读请求的 namespace:table/rowkey，查询出目标数据位于哪个 Region Server 中的哪个 Region 中。并将该 table 的 region 信息以及 meta 表的位置信息缓存在客户端的 meta cache，方便下次访问。  
3）与目标 Region Server 进行通讯；  
4）分别在 Block Cache（读缓存），MemStore 和 Store File（HFile）中查询目标数据（**同时找**，只返回符合条件的时间戳对应的数据），并将查到的所有数据进行合并。此处所有数据是指同一条数据的不同版本（time stamp）或者不同的类型（Put/Delete）。  
5）将**从文件中查询到的数据块**（Block，HFile 数据存储单元，默认大小为 64KB）缓存到 Block Cache。  
6）将合并后的最终结果返回给客户端。  
7）HBase-1.x源码, get方法

```java
package org.apache.hadoop.hbase.regionserver.HRegion;

  @Override
  public List<Cell> get(Get get, boolean withCoprocessor, long nonceGroup, long nonce)
      throws IOException {
    List<Cell> results = new ArrayList<Cell>();

    // pre-get CP hook
    if (withCoprocessor && (coprocessorHost != null)) {
      if (coprocessorHost.preGet(get, results)) {
        return results;
      }
    }
    long before = EnvironmentEdgeManager.currentTime();
    // dolly: get方法的底层也是scan
    Scan scan = new Scan(get);

    RegionScanner scanner = null;
    try {
      scanner = getScanner(scan, null, nonceGroup, nonce);
      // dolly: 从next方法往下
      scanner.next(results);
    } finally {
      if (scanner != null) {
        scanner.close();
      }
    }

    // post-get CP hook
    if (withCoprocessor && (coprocessorHost != null)) {
      coprocessorHost.postGet(get, results);
    }

    metricsUpdateForGet(results, before);

    return results;
  }

    // dolly: 太难了md, 一下子搞不定阿哈哈..
    private boolean nextInternal(List<Cell> results, ScannerContext scannerContext)
        throws IOException {
      if (!results.isEmpty()) {
        throw new IllegalArgumentException("First parameter should be an empty list");
      }
      if (scannerContext == null) {
        throw new IllegalArgumentException("Scanner context cannot be null");
      }
      RpcCallContext rpcCall = RpcServer.getCurrentCall();

      // Save the initial progress from the Scanner context in these local variables. The progress
      // may need to be reset a few times if rows are being filtered out so we save the initial
      // progress.
      int initialBatchProgress = scannerContext.getBatchProgress();
      long initialSizeProgress = scannerContext.getSizeProgress();
      long initialTimeProgress = scannerContext.getTimeProgress();

      // The loop here is used only when at some point during the next we determine
      // that due to effects of filters or otherwise, we have an empty row in the result.
      // Then we loop and try again. Otherwise, we must get out on the first iteration via return,
      // "true" if there's more data to read, "false" if there isn't (storeHeap is at a stop row,
      // and joinedHeap has no more data to read for the last row (if set, joinedContinuationRow).
      while (true) {
        // Starting to scan a new row. Reset the scanner progress according to whether or not
        // progress should be kept.
        if (scannerContext.getKeepProgress()) {
          // Progress should be kept. Reset to initial values seen at start of method invocation.
          scannerContext.setProgress(initialBatchProgress, initialSizeProgress,
            initialTimeProgress);
        } else {
          scannerContext.clearProgress();
        }

        if (rpcCall != null) {
          // If a user specifies a too-restrictive or too-slow scanner, the
          // client might time out and disconnect while the server side
          // is still processing the request. We should abort aggressively
          // in that case.
          long afterTime = rpcCall.disconnectSince();
          if (afterTime >= 0) {
            throw new CallerDisconnectedException(
                "Aborting on region " + getRegionInfo().getRegionNameAsString() + ", call " +
                    this + " after " + afterTime + " ms, since " +
                    "caller disconnected");
          }
        }

        // Let's see what we have in the storeHeap.
        Cell current = this.storeHeap.peek();

        byte[] currentRow = null;
        int offset = 0;
        short length = 0;
        if (current != null) {
          currentRow = current.getRowArray();
          offset = current.getRowOffset();
          length = current.getRowLength();
        }

        boolean stopRow = isStopRow(currentRow, offset, length);
        // When has filter row is true it means that the all the cells for a particular row must be
        // read before a filtering decision can be made. This means that filters where hasFilterRow
        // run the risk of encountering out of memory errors in the case that they are applied to a
        // table that has very large rows.
        boolean hasFilterRow = this.filter != null && this.filter.hasFilterRow();

        // If filter#hasFilterRow is true, partial results are not allowed since allowing them
        // would prevent the filters from being evaluated. Thus, if it is true, change the
        // scope of any limits that could potentially create partial results to
        // LimitScope.BETWEEN_ROWS so that those limits are not reached mid-row
        if (hasFilterRow) {
          if (LOG.isTraceEnabled()) {
            LOG.trace("filter#hasFilterRow is true which prevents partial results from being "
                + " formed. Changing scope of limits that may create partials");
          }
          scannerContext.setSizeLimitScope(LimitScope.BETWEEN_ROWS);
          scannerContext.setTimeLimitScope(LimitScope.BETWEEN_ROWS);
        }

        // Check if we were getting data from the joinedHeap and hit the limit.
        // If not, then it's main path - getting results from storeHeap.
        if (joinedContinuationRow == null) {
          // First, check if we are at a stop row. If so, there are no more results.
          if (stopRow) {
            if (hasFilterRow) {
              filter.filterRowCells(results);
            }
            return scannerContext.setScannerState(NextState.NO_MORE_VALUES).hasMoreValues();
          }

          // Check if rowkey filter wants to exclude this row. If so, loop to next.
          // Technically, if we hit limits before on this row, we don't need this call.
          if (filterRowKey(currentRow, offset, length)) {
            incrementCountOfRowsFilteredMetric(scannerContext);
            // early check, see HBASE-16296
            if (isFilterDoneInternal()) {
              return scannerContext.setScannerState(NextState.NO_MORE_VALUES).hasMoreValues();
            }
            // Typically the count of rows scanned is incremented inside #populateResult. However,
            // here we are filtering a row based purely on its row key, preventing us from calling
            // #populateResult. Thus, perform the necessary increment here to rows scanned metric
            incrementCountOfRowsScannedMetric(scannerContext);
            boolean moreRows = nextRow(scannerContext, currentRow, offset, length);
            if (!moreRows) {
              return scannerContext.setScannerState(NextState.NO_MORE_VALUES).hasMoreValues();
            }
            results.clear();
            continue;
          }

          // Ok, we are good, let's try to get some results from the main heap.
          populateResult(results, this.storeHeap, scannerContext, currentRow, offset, length);

          if (scannerContext.checkAnyLimitReached(LimitScope.BETWEEN_CELLS)) {
            if (hasFilterRow) {
              throw new IncompatibleFilterException(
                  "Filter whose hasFilterRow() returns true is incompatible with scans that must "
                      + " stop mid-row because of a limit. ScannerContext:" + scannerContext);
            }
            return true;
          }

          Cell nextKv = this.storeHeap.peek();
          stopRow = nextKv == null ||
              isStopRow(nextKv.getRowArray(), nextKv.getRowOffset(), nextKv.getRowLength());
          // save that the row was empty before filters applied to it.
          final boolean isEmptyRow = results.isEmpty();

          // We have the part of the row necessary for filtering (all of it, usually).
          // First filter with the filterRow(List).
          FilterWrapper.FilterRowRetCode ret = FilterWrapper.FilterRowRetCode.NOT_CALLED;
          if (hasFilterRow) {
            ret = filter.filterRowCellsWithRet(results);

            // We don't know how the results have changed after being filtered. Must set progress
            // according to contents of results now. However, a change in the results should not
            // affect the time progress. Thus preserve whatever time progress has been made
            long timeProgress = scannerContext.getTimeProgress();
            if (scannerContext.getKeepProgress()) {
              scannerContext.setProgress(initialBatchProgress, initialSizeProgress,
                initialTimeProgress);
            } else {
              scannerContext.clearProgress();
            }
            scannerContext.setTimeProgress(timeProgress);
            scannerContext.incrementBatchProgress(results.size());
            for (Cell cell : results) {
              scannerContext.incrementSizeProgress(CellUtil.estimatedHeapSizeOfWithoutTags(cell));
            }
          }

          if (isEmptyRow || ret == FilterWrapper.FilterRowRetCode.EXCLUDE || filterRow()) {
            incrementCountOfRowsFilteredMetric(scannerContext);
            results.clear();
            boolean moreRows = nextRow(scannerContext, currentRow, offset, length);
            if (!moreRows) {
              return scannerContext.setScannerState(NextState.NO_MORE_VALUES).hasMoreValues();
            }

            // This row was totally filtered out, if this is NOT the last row,
            // we should continue on. Otherwise, nothing else to do.
            if (!stopRow) continue;
            return scannerContext.setScannerState(NextState.NO_MORE_VALUES).hasMoreValues();
          }

          // Ok, we are done with storeHeap for this row.
          // Now we may need to fetch additional, non-essential data into row.
          // These values are not needed for filter to work, so we postpone their
          // fetch to (possibly) reduce amount of data loads from disk.
          if (this.joinedHeap != null) {
            boolean mayHaveData = joinedHeapMayHaveData(currentRow, offset, length);
            if (mayHaveData) {
              joinedContinuationRow = current;
              populateFromJoinedHeap(results, scannerContext);

              if (scannerContext.checkAnyLimitReached(LimitScope.BETWEEN_CELLS)) {
                return true;
              }
            }
          }
        } else {
          // Populating from the joined heap was stopped by limits, populate some more.
          populateFromJoinedHeap(results, scannerContext);
          if (scannerContext.checkAnyLimitReached(LimitScope.BETWEEN_CELLS)) {
            return true;
          }
        }
        // We may have just called populateFromJoinedMap and hit the limits. If that is
        // the case, we need to call it again on the next next() invocation.
        if (joinedContinuationRow != null) {
          return scannerContext.setScannerState(NextState.MORE_VALUES).hasMoreValues();
        }

        // Finally, we are done with both joinedHeap and storeHeap.
        // Double check to prevent empty rows from appearing in result. It could be
        // the case when SingleColumnValueExcludeFilter is used.
        if (results.isEmpty()) {
          incrementCountOfRowsFilteredMetric(scannerContext);
          boolean moreRows = nextRow(scannerContext, currentRow, offset, length);
          if (!moreRows) {
            return scannerContext.setScannerState(NextState.NO_MORE_VALUES).hasMoreValues();
          }
          if (!stopRow) continue;
        }

        if (stopRow) {
          return scannerContext.setScannerState(NextState.NO_MORE_VALUES).hasMoreValues();
        } else {
          return scannerContext.setScannerState(NextState.MORE_VALUES).hasMoreValues();
        }
      }
    }
```


## 读写的一致性问题

1. 图解读写流程  
   ![](https://i.imgur.com/MWu3rbJ.png)
2. MultiVersionConcurrencyControl

- 参考: [HBase MVCC实现流程](https://www.jianshu.com/p/86246d8bee24)
- HBase-1.x源码

```java
/**
 * Manages the read/write consistency. This provides an interface for readers to determine what
 * entries to ignore, and a mechanism for writers to obtain new write numbers, then "commit"
 * the new writes for readers to read (thus forming atomic transactions).
 */
package org.apache.hadoop.hbase.regionserver.MultiVersionConcurrencyControl;

  /**
   * Start a write transaction. Create a new {@link WriteEntry} with a new write number and add it
   * to our queue of ongoing writes. Return this WriteEntry instance. To complete the write
   * transaction and wait for it to be visible, call {@link #completeAndWait(WriteEntry)}. If the
   * write failed, call {@link #complete(WriteEntry)} so we can clean up AFTER removing ALL trace of
   * the failed write transaction.
   * <p>
   * The {@code action} will be executed under the lock which means it can keep the same order with
   * mvcc.
   * @see #complete(WriteEntry)
   * @see #completeAndWait(WriteEntry)
   */
  public WriteEntry begin(Runnable action) {
    synchronized (writeQueue) {
      long nextWriteNumber = writePoint.incrementAndGet();
      WriteEntry e = new WriteEntry(nextWriteNumber);
      writeQueue.add(e);
      action.run();
      return e;
    }
  }
```

## MemStore Flush

- [HBase – Memstore Flush深度解析](http://hbasefly.com/2016/03/23/hbase-memstore-flush/)
- [Hbase Memstore 读写及 flush 源码分析](https://cloud.tencent.com/developer/article/1005758)

1. Flush触发条件

- 大小达到刷写阀值:  
  (1) 当某个 memstore 的大小达到了  
  hbase.hregion.memstore.flush.size （默认值 128M）  
  时，其所在 region 的所有 memstore 都会刷写。  
  (2) 当 memstore 的大小达到了  
  hbase.hregion.memstore.flush.size （默认值 128M）× hbase.hregion.memstore.block.multiplier （默认值 4）  
  时，会阻止继续往该 memstore 写数据。

- 整个RegionServer的memstore总和达到阀值:  
  (1) 当 region server 中 memstore 的总大小达到  
  java_heapsize × hbase.regionserver.global.memstore.size （默认值 0.4）× hbase.regionserver.global.memstore.size.lower.limit （默认值 0.95）  
  时，region 会按照其所有 memstore 的大小顺序（由大到小）依次进行刷写。直到 region server
  中所有 memstore 的总大小减小到上述值以下。  
  (2) 当 region server 中 memstore 的总大小达到  
  java_heapsize × hbase.regionserver.global.memstore.size （默认值 0.4）  
  时，会阻止继续往所有的 memstore 写数据。

- 到达自动刷写的时间，也会触发 memstore flush。自动刷新的时间间隔由该属性进行配置  
  hbase.regionserver.optionalcacheflushinterval （默认 1 小时）。

- 当 WAL 文件的数量超过 hbase.regionserver.max.logs，region 会按照时间顺序依次进行刷写，直到 WAL 文件数量减小到 hbase.regionserver.max.log 以下（该属性名已经废弃，现无需手动设置，最大值为 32）。

- 手动触发flush

2. HBase-1.x源码

```java
package org.apache.hadoop.hbase.regionserver.HRegion;

  /**
   * Flush the cache.
   *
   * When this method is called the cache will be flushed unless:
   * <ol>
   *   <li>the cache is empty</li>
   *   <li>the region is closed.</li>
   *   <li>a flush is already in progress</li>
   *   <li>writes are disabled</li>
   * </ol>
   *
   * <p>This method may block for some time, so it should not be called from a
   * time-sensitive thread.
   * @param forceFlushAllStores whether we want to flush all stores
   * @param writeFlushRequestWalMarker whether to write the flush request marker to WAL
   * @return whether the flush is success and whether the region needs compacting
   *
   * @throws IOException general io exceptions
   * @throws DroppedSnapshotException Thrown when replay of wal is required
   * because a Snapshot was not properly persisted. The region is put in closing mode, and the
   * caller MUST abort after this.
   */
  public FlushResult flushcache(boolean forceFlushAllStores, boolean writeFlushRequestWalMarker)
      throws IOException {
    // fail-fast instead of waiting on the lock
    if (this.closing.get()) {
      String msg = "Skipping flush on " + this + " because closing";
      LOG.debug(msg);
      return new FlushResultImpl(FlushResult.Result.CANNOT_FLUSH, msg, false);
    }
    MonitoredTask status = TaskMonitor.get().createStatus("Flushing " + this);
    status.setStatus("Acquiring readlock on region");
    // block waiting for the lock for flushing cache
    // dolly: 加锁
    lock.readLock().lock();
    try {
      if (this.closed.get()) {
        String msg = "Skipping flush on " + this + " because closed";
        LOG.debug(msg);
        status.abort(msg);
        return new FlushResultImpl(FlushResult.Result.CANNOT_FLUSH, msg, false);
      }
      if (coprocessorHost != null) {
        status.setStatus("Running coprocessor pre-flush hooks");
        coprocessorHost.preFlush();
      }
      // TODO: this should be managed within memstore with the snapshot, updated only after flush
      // successful
      if (numMutationsWithoutWAL.get() > 0) {
        numMutationsWithoutWAL.set(0);
        dataInMemoryWithoutWAL.set(0);
      }
      // dolly: 获得同步状态
      synchronized (writestate) {
        if (!writestate.flushing && writestate.writesEnabled) {
          this.writestate.flushing = true;
        } else {
          if (LOG.isDebugEnabled()) {
            LOG.debug("NOT flushing memstore for region " + this
                + ", flushing=" + writestate.flushing + ", writesEnabled="
                + writestate.writesEnabled);
          }
          String msg = "Not flushing since "
              + (writestate.flushing ? "already flushing"
              : "writes not enabled");
          status.abort(msg);
          return new FlushResultImpl(FlushResult.Result.CANNOT_FLUSH, msg, false);
        }
      }

      try {
        // dolly: 获得需要Flush的Stores
        Collection<Store> specificStoresToFlush =
            forceFlushAllStores ? stores.values() : flushPolicy.selectStoresToFlush();
        // dolly: 具体的Flush事务
        FlushResult fs = internalFlushcache(specificStoresToFlush,
          status, writeFlushRequestWalMarker);

        if (coprocessorHost != null) {
          status.setStatus("Running post-flush coprocessor hooks");
          coprocessorHost.postFlush();
        }

        // dolly: 标记为成功
        status.markComplete("Flush successful");
        return fs;
      } finally {
        synchronized (writestate) {
          writestate.flushing = false;
          this.writestate.flushRequested = false;
          // dolly: 唤醒其他线程
          writestate.notifyAll();
        }
      }
    } finally {
      // dolly: 释放锁
      lock.readLock().unlock();
      status.cleanup();
    }
  }

  /**
   * Flush the memstore. Flushing the memstore is a little tricky. We have a lot
   * of updates in the memstore, all of which have also been written to the wal.
   * We need to write those updates in the memstore out to disk, while being
   * able to process reads/writes as much as possible during the flush
   * operation.
   * <p>
   * This method may block for some time. Every time you call it, we up the
   * regions sequence id even if we don't flush; i.e. the returned region id
   * will be at least one larger than the last edit applied to this region. The
   * returned id does not refer to an actual edit. The returned id can be used
   * for say installing a bulk loaded file just ahead of the last hfile that was
   * the result of this flush, etc.
   *
   * @param wal
   *          Null if we're NOT to go via wal.
   * @param myseqid
   *          The seqid to use if <code>wal</code> is null writing out flush
   *          file.
   * @param storesToFlush
   *          The list of stores to flush.
   * @return object describing the flush's state
   * @throws IOException
   *           general io exceptions
   * @throws DroppedSnapshotException
   *           Thrown when replay of wal is required because a Snapshot was not
   *           properly persisted.
   */
  protected FlushResult internalFlushcache(final WAL wal, final long myseqid,
      final Collection<Store> storesToFlush, MonitoredTask status, boolean writeFlushWalMarker)
          throws IOException {
    // dolly: 类似两阶段提交方式
    // dolly: 准备快照和中间临时文件
    PrepareFlushResult result
      = internalPrepareFlushCache(wal, myseqid, storesToFlush, status, writeFlushWalMarker);
    if (result.result == null) {
      // dolly: 把快照刷到磁盘, 保存在临时文件夹下; 再把临时文件夹移动到文件根路径
      return internalFlushCacheAndCommit(wal, status, result, storesToFlush);
    } else {
      return result.result; // early exit due to failure from prepare stage
    }
  }
```

## StoreFile Compaction ##

0. 参考资料

- [HBase Compaction的前生今世－身世之旅](http://hbasefly.com/2016/07/13/hbase-compaction-1/)
- [HBase Compaction的前生今世－改造之路](http://hbasefly.com/2016/07/25/hbase-compaction-2/)
- [hbase源码系列（十四）Compact和Split](https://www.cnblogs.com/cenyuhai/p/3746473.html)

1. 由于memstore每次刷写都会生成一个新的HFile，且同一个字段的不同版本（timestamp）和不同类型（Put/Delete）有可能会分布在不同的HFile中，因此查询时需要遍历所有的HFile。为了减少 HFile 的个数，以及清理掉过期和删除的数据，会进行 StoreFile Compaction。  
2. 作用:  
a. 合并文件  
b. 清除删除、过期、多余版本的数据  
c. 提高读写数据的效率  
3. 两种compaction的区别:  
(1) Minor Compaction会将临近的若干个较小的 HFile 合并成一个较大的 HFile，但**不会**清理过期和删除的数据。  
(2) Major Compaction 会将一个 Store 下的所有的 HFile 合并成一个大 HFile，并且**会**清理掉过期
和删除的数据。
3. compaction触发时机：  
a. 当数据块达到3块，HMaster触发合并操作，Region将数据块加载到本地，进行合并  
b. CompactionChecker线程，周期轮询  
c. 手动触发compact  
5. HBase-1.x源码

```java
package org.apache.hadoop.hbase.regionserver.CompactSplitThread;

  /**
   * @param r region store belongs to
   * @param s Store to request compaction on
   * @param why Why compaction requested -- used in debug messages
   * @param priority override the default priority (NO_PRIORITY == decide)
   * @param request custom compaction request. Can be <tt>null</tt> in which case a simple
   *          compaction will be used.
   */
  private synchronized CompactionRequest requestCompactionInternal(final Region r, final Store s,
      final String why, int priority, CompactionRequest request, boolean selectNow, User user)
          throws IOException {
    if (this.server.isStopped()
        || (r.getTableDesc() != null && !r.getTableDesc().isCompactionEnabled())) {
      return null;
    }

    CompactionContext compaction = null;
    if (selectNow) {
      compaction = selectCompaction(r, s, priority, request, user);
      if (compaction == null) return null; // message logged inside
    }

    // We assume that most compactions are small. So, put system compactions into small
    // pool; we will do selection there, and move to large pool if necessary.
    // dolly: 自定义了一个线程池
    ThreadPoolExecutor pool = (selectNow && s.throttleCompaction(compaction.getRequest().getSize()))
      ? longCompactions : shortCompactions;
    pool.execute(new CompactionRunner(s, r, compaction, pool, user));
    if (LOG.isDebugEnabled()) {
      String type = (pool == shortCompactions) ? "Small " : "Large ";
      LOG.debug(type + "Compaction requested: " + (selectNow ? compaction.toString() : "system")
          + (why != null && !why.isEmpty() ? "; Because: " + why : "") + "; " + this);
    }
    return selectNow ? compaction.getRequest() : null;
  }

package org.apache.hadoop.hbase.regionserver.compaction.SortedCompactionPolicy;

  /**
   * @param candidateFiles candidate files, ordered from oldest to newest by seqId. We rely on
   *   DefaultStoreFileManager to sort the files by seqId to guarantee contiguous compaction based
   *   on seqId for data consistency.
   * @return subset copy of candidate list that meets compaction criteria
   */
  public CompactionRequest selectCompaction(Collection<StoreFile> candidateFiles,
      final List<StoreFile> filesCompacting, final boolean isUserCompaction,
      final boolean mayUseOffPeak, final boolean forceMajor) throws IOException {
    // Preliminary compaction subject to filters
    ArrayList<StoreFile> candidateSelection = new ArrayList<StoreFile>(candidateFiles);
    // Stuck and not compacting enough (estimate). It is not guaranteed that we will be
    // able to compact more if stuck and compacting, because ratio policy excludes some
    // non-compacting files from consideration during compaction (see getCurrentEligibleFiles).
    int futureFiles = filesCompacting.isEmpty() ? 0 : 1;
    boolean mayBeStuck = (candidateFiles.size() - filesCompacting.size() + futureFiles)
        >= storeConfigInfo.getBlockingFileCount();

    candidateSelection = getCurrentEligibleFiles(candidateSelection, filesCompacting);
    LOG.debug("Selecting compaction from " + candidateFiles.size() + " store files, " +
        filesCompacting.size() + " compacting, " + candidateSelection.size() +
        " eligible, " + storeConfigInfo.getBlockingFileCount() + " blocking");

    // If we can't have all files, we cannot do major anyway
    boolean isAllFiles = candidateFiles.size() == candidateSelection.size();
    if (!(forceMajor && isAllFiles)) {
      //dolly: minor compact跳过大文件
      candidateSelection = skipLargeFiles(candidateSelection, mayUseOffPeak);
      isAllFiles = candidateFiles.size() == candidateSelection.size();
    }

    // Try a major compaction if this is a user-requested major compaction,
    // or if we do not have too many files to compact and this was requested as a major compaction
    boolean isTryingMajor = (forceMajor && isAllFiles && isUserCompaction)
        || (((forceMajor && isAllFiles) || shouldPerformMajorCompaction(candidateSelection))
          && (candidateSelection.size() < comConf.getMaxFilesToCompact()));
    // Or, if there are any references among the candidates.
    boolean isAfterSplit = StoreUtils.hasReferences(candidateSelection);

    CompactionRequest result = createCompactionRequest(candidateSelection,
      isTryingMajor || isAfterSplit, mayUseOffPeak, mayBeStuck);

    ArrayList<StoreFile> filesToCompact = Lists.newArrayList(result.getFiles());
    removeExcessFiles(filesToCompact, isUserCompaction, isTryingMajor);
    result.updateFiles(filesToCompact);

    isAllFiles = (candidateFiles.size() == filesToCompact.size());
    result.setOffPeak(!filesToCompact.isEmpty() && !isAllFiles && mayUseOffPeak);
    result.setIsMajor(isTryingMajor && isAllFiles, isAllFiles);

    return result;
  }

package org.apache.hadoop.hbase.regionserver.compactions.RatioBasedCompactionPolicy;

  @Override
  protected CompactionRequest createCompactionRequest(ArrayList<StoreFile> candidateSelection,
    boolean tryingMajor, boolean mayUseOffPeak, boolean mayBeStuck) throws IOException {
    if (!tryingMajor) {
      //dolly: 过滤掉Bulkload进来的文件
      candidateSelection = filterBulk(candidateSelection);
      //dolly: 过滤掉大小不满足的文件
      candidateSelection = applyCompactionPolicy(candidateSelection, mayUseOffPeak, mayBeStuck);
      //dolly: 检查文件数是否满足最小的要求
      candidateSelection = checkMinFilesCriteria(candidateSelection,
        comConf.getMinFilesToCompact());
    }
    return new CompactionRequest(candidateSelection);
  }
```



## Region Split ##

0. 参考资料  
- [HBase原理 – 所有Region切分的细节都在这里了](http://hbasefly.com/2017/08/27/hbase-split/)
- [hbase源码系列（十四）Compact和Split](https://cloud.tencent.com/developer/article/1048004)

1. 默认情况下，每个 Table 起初只有一个 Region，随着数据的不断写入，Region 会自动进行拆分。刚拆分时，两个子 Region 都位于当前的 Region Server，但处于负载均衡的考虑，HMaster 有可能会将某个 Region 转移给其他的 Region Server。  
   Region Split 时机：  
   (1) 当 1 个region中的某个Store下所有StoreFile的总大小超过hbase.hregion.max.filesize，该 Region 就会进行拆分（0.94 版本之前）。  
   (2) 当 1 个 region 中 的某 个 Store 下所有 StoreFile 的总大小超过 Min(R^2 × "hbase.hregion.memstore.flush.size", hbase.hregion.max.filesize")，该 Region 就会进行拆分，其中 R 为当前 Region Server 中属于该 Table 的个数（0.94 版本之后）。  
   (3) 每次切分**不一定都一样大**：涉及到**midKey**查询
2. HBase-1.x源码

```java
package org.apache.hadoop.hbase.regionserver.CompactSplitThread;

  public synchronized boolean requestSplit(final Region r) {
    // don't split regions that are blocking
    if (shouldSplitRegion() && ((HRegion)r).getCompactPriority() >= Store.PRIORITY_USER) {
      // dolly: midkey为切分点
      byte[] midKey = ((HRegion)r).checkSplit();
      if (midKey != null) {
        requestSplit(r, midKey);
        return true;
      }
    }
    return false;
  }

package org.apache.hadoop.hbase.regionserver.RegionSplitPolicy;

  /**
   * @return the key at which the region should be split, or null
   * if it cannot be split. This will only be called if shouldSplit
   * previously returned true.
   */
  protected byte[] getSplitPoint() {
    byte[] explicitSplitPoint = this.region.getExplicitSplitPoint();
    if (explicitSplitPoint != null) {
      return explicitSplitPoint;
    }
    List<Store> stores = region.getStores();

    byte[] splitPointFromLargestStore = null;
    long largestStoreSize = 0;
    for (Store s : stores) {
      byte[] splitPoint = s.getSplitPoint();
      // Store also returns null if it has references as way of indicating it is not splittable
      long storeSize = s.getSize();
      // dolly: 找出最大的HStore，然后通过它来找这个分裂点，最大的文件的中间点
      if (splitPoint != null && largestStoreSize < storeSize) {
        splitPointFromLargestStore = splitPoint;
        largestStoreSize = storeSize;
      }
    }

    return splitPointFromLargestStore;
  }
```

## rowkey设计 ##
1. 原则:  
a. rowkey长度原则  
b. rowkey散列原则  
c. rowkey唯一原则
2. 方案:  
a. 生成随机数、hash、散列值  
b. 字符串反转  
c. 字符串拼接、加盐  
d. 时间戳反转
3. 热点问题  
3.1 产生热点问题的原因：  
	(1)	hbase的中的数据是按照字典序排序的，当大量连续的rowkey集中写在个别的region，各个region之间数据分布不均衡  
	(2)	创建表时没有提前预分区，创建的表默认只有一个region，大量的数据写入当前region  
	(3)	创建表已经提前预分区，但是设计的rowkey没有规律可循  
3.2 热点问题的解决方案：  
	(1)	随机数 + 业务主键，如果想让最近的数据快速get到，可以将时间戳加上  
	(2)	rowkey设计越短越好，不要超过10~100个字节  
	(3)	映射regionNo，这样既可以让数据均匀分布到各个region中，同时可以根据startkey和endkey可以get到同一批数据  

## CMS GC

0. 参考资料
- [HBase GC的前生今世 – 身世篇](http://hbasefly.com/2016/05/21/hbase-gc-1/)
- [HBase GC的前生今世 – 演进篇](http://hbasefly.com/2016/05/29/hbase-gc-2/)
- [HBase最佳实践－CMS GC调优](http://hbasefly.com/2016/05/29/hbase-gc-2/)

1. CMS失效  
![](https://i.imgur.com/wrEQ8uC.png)
2. 调优经验
- HBase 操作过程中需要大量的内存开销，毕竟 Table 是可以缓存在内存中的，一般会分配整个可用内存的 70% 给 HBase 的 Java 堆。但是不建议分配非常大的堆内存，因为 FGC 过程持续太久会导致 RegionServer 处于长期不可用状态，一般 16~48G 内存就可以了，如果因为框架占用内存过高导致系统内存不足，框架一样会被系统服务拖死。
- RegionServer内存大于32GB，建议使用G1GC策略；一般情况ParallelGC + CMS即可

