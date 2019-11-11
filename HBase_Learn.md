# HBase原理和源码 #
2019/9/18 23:19:40  

## 参考资料 ##
- [有态度的HBase/Spark/BigData](http://hbasefly.com/?vilqlm=bnem43&xgrony=vo0822)  
- [Openinx Blog](http://openinx.github.io/)  
- [岑玉海-hbase源码系列](https://www.cnblogs.com/cenyuhai/tag/hbase%E6%BA%90%E7%A0%81%E7%B3%BB%E5%88%97/)  
- 《尚硅谷大数据技术之HBase》
- 《HBase原理与实践》
- 《HBase不睡觉书》

## 存储架构 ##
![](https://i.imgur.com/CUXyzuj.png)

## 写流程 ##
![](https://i.imgur.com/dTfHM1P.png)  
1）Client 先访问 zookeeper，获取 hbase:meta 表位于哪个 Region Server。  
2）访问对应的 Region Server，获取 hbase:meta 表，根据读请求的 namespace:table/rowkey，查询出目标数据位于哪个 Region Server 中的哪个 Region 中。并将该 table 的 region 信息以及 meta 表的位置信息缓存在客户端的 meta cache，方便下次访问。  
3）与目标 Region Server 进行通讯；  
4）将数据顺序写入（追加）到 WAL；  
5）将数据写入对应的 MemStore，数据会在 MemStore 进行排序；  
6）向客户端发送 ack；  
7）等达到 MemStore 的刷写时机后，将数据刷写到 HFile。  
8）HBase-1.3.1源码
 ```
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
	  // dolly: 获取JUC中的锁, 保证读写分离; 下面这一大堆是需要考虑到的异常情况
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
      now = EnvironmentEdgeManager.currentTime();
      byte[] byteNow = Bytes.toBytes(now);

      // Nothing to put/delete -- an exception in the above such as NoSuchColumnFamily?
      if (numReadyToWrite <= 0) return 0L;

      // We've now grabbed as many mutations off the list as we can

      // ------------------------------------
      // STEP 2. Update any LATEST_TIMESTAMP timestamps
      // ----------------------------------
      for (int i = firstIndex; !isInReplay && i < lastIndexExclusive; i++) {
        // skip invalid
        if (batchOp.retCodeDetails[i].getOperationStatusCode()
            != OperationStatusCode.NOT_RUN) continue;

        Mutation mutation = batchOp.getMutation(i);
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
	  // dolly: 在内存中, 把WAL的修改对象追加到WAL之后, 但不写到HDFS
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
	  // dolly: 执行协处理器相关操作
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
9）HBase-2.X源码
 ```
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
	  // dolly: 和1.X不同, 在内存中构建完WAL后会直接写到HDFS上
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

## MemStore Flush ##
1. 大小达到刷写阀值:  
(1) 当某个 memstore 的大小达到了  
hbase.hregion.memstore.flush.size （默认值 128M）  
时，其所在 region 的所有 memstore 都会刷写。  
(2) 当 memstore 的大小达到了  
hbase.hregion.memstore.flush.size （默认值 128M）× hbase.hregion.memstore.block.multiplier （默认值 4）  
时，会阻止继续往该 memstore 写数据。
2. 整个RegionServer的memstore总和达到阀值:  
(1) 当 region server 中 memstore 的总大小达到  
java_heapsize × hbase.regionserver.global.memstore.size （默认值 0.4）× hbase.regionserver.global.memstore.size.lower.limit （默认值 0.95）  
时，region 会按照其所有 memstore 的大小顺序（由大到小）依次进行刷写。直到 region server
中所有 memstore 的总大小减小到上述值以下。  
(2) 当 region server 中 memstore 的总大小达到  
java_heapsize × hbase.regionserver.global.memstore.size （默认值 0.4）  
时，会阻止继续往所有的 memstore 写数据。
3. 到达自动刷写的时间，也会触发 memstore flush。自动刷新的时间间隔由该属性进行配置  
hbase.regionserver.optionalcacheflushinterval （默认 1 小时）。
4. 当 WAL 文件的数量超过 hbase.regionserver.max.logs，region 会按照时间顺序依次进行刷写，直到 WAL 文件数量减小到 hbase.regionserver.max.log 以下（该属性名已经废弃，现无需手动设置，最大值为 32）。
5. 手动触发flush

## 读流程 ##
![](https://i.imgur.com/rlOHKlj.png)  
1）Client 先访问 zookeeper，获取 hbase:meta 表位于哪个 Region Server。  
2）访问对应的 Region Server，获取 hbase:meta 表，根据读请求的 namespace:table/rowkey，查询出目标数据位于哪个 Region Server 中的哪个 Region 中。并将该 table 的 region 信息以及 meta 表的位置信息缓存在客户端的 meta cache，方便下次访问。  
3）与目标 Region Server 进行通讯；  
4）分别在 Block Cache（读缓存），MemStore 和 Store File（HFile）中查询目标数据（**同时找**，只返回符合条件的时间戳对应的数据），并将查到的所有数据进行合并。此处所有数据是指同一条数据的不同版本（time stamp）或者不同的类型（Put/Delete）。  
5）将**从文件中查询到的数据块**（Block，HFile 数据存储单元，默认大小为 64KB）缓存到 Block Cache。  
6）将合并后的最终结果返回给客户端。

## StoreFile Compaction ##
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

## Region Split ##
默认情况下，每个 Table 起初只有一个 Region，随着数据的不断写入，Region 会自动进行拆分。刚拆分时，两个子 Region 都位于当前的 Region Server，但处于负载均衡的考虑，HMaster 有可能会将某个 Region 转移给其他的 Region Server。  
Region Split 时机：  
(1) 当 1 个region中的某个Store下所有StoreFile的总大小超过hbase.hregion.max.filesize，该 Region 就会进行拆分（0.94 版本之前）。  
(2) 当 1 个 region 中 的某 个 Store 下所有 StoreFile 的总大小超过 Min(R^2 × "hbase.hregion.memstore.flush.size", hbase.hregion.max.filesize")，该 Region 就会进行拆分，其中 R 为当前 Region Server 中属于该 Table 的个数（0.94 版本之后）。
(3) 每次切分不一定都一样大：涉及到**middleKey**查询

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
(1)	随机数+业务主键，如果想让最近的数据快速get到，可以将时间戳加上  
(2)	rowkey设计越短越好，不要超过10~100个字节  
(3)	映射regionNo，这样既可以让数据均匀分布到各个region中，同时可以根据startkey和endkey可以get到同一批数据  
