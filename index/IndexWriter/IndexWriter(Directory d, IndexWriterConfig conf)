  public IndexWriter(Directory d, IndexWriterConfig conf) throws IOException {
    //数据域enableTestPoints(private final boolean，used only for testing)
    enableTestPoints = isEnableTestPoints();
    // prevent reuse by other instances
    conf.setIndexWriter(this);
    //数据域config(private final LiveIndexWriterConfig, it's saved only to allow users to query an IndexWriter settings)
    //final class IndexWriterConfig(子,conf) extends LiveIndexWriterConfig（父,config）
    config = conf;//upcasting
    //调用子类IndexWriterConfig的getInfoStream()方法——return infoStream;
    infoStream = config.getInfoStream();
    //getSoftDeletesFailed()——return softDeletesField;
    //LiveIndexWriterConfig::softDeletesField = null,
    //IndexWriterConfig::setSoftDeletesField(String softDeletesField)——this.softDeletesField=softDeletesField;
    softDeletesEnabled = config.getSoftDeletesField() != null;
    // obtain the write.lock. If the user configured a timeout,
    // we wrap with a sleeper and this might take some time.
    writeLock = d.obtainLock(WRITE_LOCK_NAME);
    
    boolean success = false;
    try {
      directoryOrig = d;
      /** 
       * Class LockValidatingDirectoryWrapper makes a best-effort check that a provided {@link Lock}
       * is valid before any destructive filesystem operation.
       * LockValidatingDirectoryWrapper(final) ——> FilterDirectory(abstract) ——> Directory
       */
      directory = new LockValidatingDirectoryWrapper(d, writeLock);//upcasting

      analyzer = config.getAnalyzer();
      mergeScheduler = config.getMergeScheduler();
      mergeScheduler.setInfoStream(infoStream);
      codec = config.getCodec();
      OpenMode mode = config.getOpenMode();
      final boolean indexExists;
      final boolean create;
      if (mode == OpenMode.CREATE) {
        indexExists = DirectoryReader.indexExists(directory);
        create = true;
      } else if (mode == OpenMode.APPEND) {
        indexExists = true;
        create = false;
      } else {
        // CREATE_OR_APPEND - create only if an index does not exist
        indexExists = DirectoryReader.indexExists(directory);
        create = !indexExists;
      }

      // If index is too old, reading the segments will throw
      // IndexFormatTooOldException.
      //TODO:存疑
      String[] files = directory.listAll();

      /** IndexCommit
       * <p>Expert: represents a single commit into an index as seen by the
       * {@link IndexDeletionPolicy} or {@link IndexReader}.</p>
       *
       * <p> Changes to the content of an index are made visible
       * only after the writer who made that change commits by
       * writing a new segments file
       * (<code>segments_N</code>). This point in time, when the
       * action of writing of a new segments file to the directory
       * is completed, is an index commit.</p>
       *
       * <p>Each index commit point has a unique segments file
       * associated with it. The segments file associated with a
       * later index commit point would have a larger N.</p>
       *
       * @lucene.experimental
      */

      // Set up our initial SegmentInfos:
      IndexCommit commit = config.getIndexCommit();
      StandardDirectoryReader reader;
      if (commit == null) {
        reader = null;
      } else {
        reader = commit.getReader();
      }

      if (create) {
        if (config.getIndexCommit() != null) {
          // We cannot both open from a commit point and create:
          if (mode == OpenMode.CREATE) {
            throw new IllegalArgumentException("cannot use IndexWriterConfig.setIndexCommit() with OpenMode.CREATE");
          } else {
            throw new IllegalArgumentException("cannot use IndexWriterConfig.setIndexCommit() when index has no commit");
          }
        }

        // Try to read first.  This is to allow create
        // against an index that's currently open for
        // searching.  In this case we write the next
        // segments_N file with no segments:
        // 总之，将segmentInfos置为LatestCommit版本
        final SegmentInfos sis = new SegmentInfos(config.getIndexCreatedVersionMajor());
        if (indexExists) {
          final SegmentInfos previous = SegmentInfos.readLatestCommit(directory);
          sis.updateGenerationVersionAndCounter(previous);
        }
        segmentInfos = sis;
        rollbackSegments = segmentInfos.createBackupSegmentInfos();

        // Record that we have a change (zero out all
        // segments) pending:
        changed();

      } else if (reader != null) {
        // Init from an existing already opened NRT or non-NRT reader:
      
        if (reader.directory() != commit.getDirectory()) {
          throw new IllegalArgumentException("IndexCommit's reader must have the same directory as the IndexCommit");
        }

        if (reader.directory() != directoryOrig) {
          throw new IllegalArgumentException("IndexCommit's reader must have the same directory passed to IndexWriter");
        }

        if (reader.segmentInfos.getLastGeneration() == 0) {  
          // TODO: maybe we could allow this?  It's tricky...
          throw new IllegalArgumentException("index must already have an initial commit to open from reader");
        }

        // Must clone because we don't want the incoming NRT reader to "see" any changes this writer now makes:
        segmentInfos = reader.segmentInfos.clone();

        SegmentInfos lastCommit;
        try {
          lastCommit = SegmentInfos.readCommit(directoryOrig, segmentInfos.getSegmentsFileName());
        } catch (IOException ioe) {
          throw new IllegalArgumentException("the provided reader is stale: its prior commit file \"" + segmentInfos.getSegmentsFileName() + "\" is missing from index");
        }

        if (reader.writer != null) {

          // The old writer better be closed (we have the write lock now!):
          assert reader.writer.closed;

          // In case the old writer wrote further segments (which we are now dropping),
          // update SIS metadata so we remain write-once:
          segmentInfos.updateGenerationVersionAndCounter(reader.writer.segmentInfos);
          lastCommit.updateGenerationVersionAndCounter(reader.writer.segmentInfos);
        }

        rollbackSegments = lastCommit.createBackupSegmentInfos();
      } else {
        // Init from either the latest commit point, or an explicit prior commit point:

        String lastSegmentsFile = SegmentInfos.getLastCommitSegmentsFileName(files);
        if (lastSegmentsFile == null) {
          throw new IndexNotFoundException("no segments* file found in " + directory + ": files: " + Arrays.toString(files));
        }

        // Do not use SegmentInfos.read(Directory) since the spooky
        // retrying it does is not necessary here (we hold the write lock):
        segmentInfos = SegmentInfos.readCommit(directoryOrig, lastSegmentsFile);

        if (commit != null) {
          // Swap out all segments, but, keep metadata in
          // SegmentInfos, like version & generation, to
          // preserve write-once.  This is important if
          // readers are open against the future commit
          // points.
          if (commit.getDirectory() != directoryOrig) {
            throw new IllegalArgumentException("IndexCommit's directory doesn't match my directory, expected=" + directoryOrig + ", got=" + commit.getDirectory());
          }
          
          SegmentInfos oldInfos = SegmentInfos.readCommit(directoryOrig, commit.getSegmentsFileName());
          segmentInfos.replace(oldInfos);
          changed();

          if (infoStream.isEnabled("IW")) {
            infoStream.message("IW", "init: loaded commit \"" + commit.getSegmentsFileName() + "\"");
          }
        }

        rollbackSegments = segmentInfos.createBackupSegmentInfos();
      }



      commitUserData = new HashMap<>(segmentInfos.getUserData()).entrySet();

      pendingNumDocs.set(segmentInfos.totalMaxDoc());

      // start with previous field numbers, but new FieldInfos
      // NOTE: this is correct even for an NRT reader because we'll pull FieldInfos even for the un-committed segments:
      globalFieldNumberMap = getFieldNumberMap();

      validateIndexSort();

      config.getFlushPolicy().init(config);
      bufferedUpdatesStream = new BufferedUpdatesStream(infoStream);
      docWriter = new DocumentsWriter(flushNotifications, segmentInfos.getIndexCreatedVersionMajor(), pendingNumDocs,
          enableTestPoints, this::newSegmentName,
          config, directoryOrig, directory, globalFieldNumberMap);
      readerPool = new ReaderPool(directory, directoryOrig, segmentInfos, globalFieldNumberMap,
          bufferedUpdatesStream::getCompletedDelGen, infoStream, conf.getSoftDeletesField(), reader, config.getReaderAttributes());
      if (config.getReaderPooling()) {
        readerPool.enableReaderPooling();
      }
      // Default deleter (for backwards compatibility) is
      // KeepOnlyLastCommitDeleter:

      // Sync'd is silly here, but IFD asserts we sync'd on the IW instance:
      synchronized(this) {
        deleter = new IndexFileDeleter(files, directoryOrig, directory,
                                       config.getIndexDeletionPolicy(),
                                       segmentInfos, infoStream, this,
                                       indexExists, reader != null);

        // We incRef all files when we return an NRT reader from IW, so all files must exist even in the NRT case:
        assert create || filesExist(segmentInfos);
      }

      if (deleter.startingCommitDeleted) {
        // Deletion policy deleted the "head" commit point.
        // We have to mark ourself as changed so that if we
        // are closed w/o any further changes we write a new
        // segments_N file.
        changed();
      }

      if (reader != null) {
        // We always assume we are carrying over incoming changes when opening from reader:
        segmentInfos.changed();
        changed();
      }

      if (infoStream.isEnabled("IW")) {
        infoStream.message("IW", "init: create=" + create + " reader=" + reader);
        messageState();
      }

      success = true;

    } finally {
      if (!success) {
        if (infoStream.isEnabled("IW")) {
          infoStream.message("IW", "init: hit exception on init; releasing write lock");
        }
        IOUtils.closeWhileHandlingException(writeLock);
        writeLock = null;
      }
    }
  }

  /** Confirms that the incoming index sort (if any) matches the existing index sort (if any).  */
  private void validateIndexSort() {
    Sort indexSort = config.getIndexSort();
    if (indexSort != null) {
      for(SegmentCommitInfo info : segmentInfos) {
        Sort segmentIndexSort = info.info.getIndexSort();
        if (segmentIndexSort == null || isCongruentSort(indexSort, segmentIndexSort) == false) {
          throw new IllegalArgumentException("cannot change previous indexSort=" + segmentIndexSort + " (from segment=" + info + ") to new indexSort=" + indexSort);
        }
      }
    }
  }
