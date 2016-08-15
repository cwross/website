+++
type = "blog"
author = "jvff"
title = "Ext3 Journal Implementation"
date = "2010-08-05T00:49:14.000Z"
tags = ["ext3", "journal", "gsoc", "gsoc2010"]
+++

The fundamental parts of the journal code are finished. Although they still need more testing, and they can change as more of the Ext3 code is written, they are ready for supporting the first steps in write support for ext2 and ext3 volumes. This blog post explains the code, and how it is organized to handle ext2 and ext3 volumes.
<!--break-->
The first thing to notice is that a journal only exists in Ext3 volumes, and therefore Ext2 volumes should not have their operations logged. However, having support for transactions makes it easier to handle error checking in file system operations. Because one operation (in most cases) is composed of many sub-operations, a transaction groups all of them to represent one operation, as if it was an atomic operation. The transaction then allows all operations to be "simulated" before actually applying them on the file system. Therefore, we can just do the sub operations in order, and if one of them fails, we can fail the transaction with the tranquillity that all the other sub operations will be undone.

Most of this facility comes from the block cache's support for transactions. A transaction is only applied after it has been closed. The block cache is responsible for managing the transactions, having support for starting, cancelling, finishing (closing), and flushing a transaction. There is also support for sub transactions, which could be described as another independent transaction that is started while the "parent" transaction still hasn't been flushed to disk. Therefore, all of the transaction support comes mostly for free, and the only things that remain is to listen to transaction states when we need to do something that the block cache doesn't do automatically (for example restore memory buffers when transactions fail) and actually map the transaction to the disk journal.

There are three main classes related to the journal code. The first class is the Journal class, which contains most of the code. It performs the actual mapping of the disk journal, mostly through the _WriteTransactionToLog function. The other two classes are sub-classes of the Journal class, adding or removing features for specialization. The first sub-class is the InodeJournal, which represents an on disk journal inside an inode (the simplest and most common form of an ext3 journal). The main method that this class overrides is MapBlock, which passes the logical block number through Inode::FindBlock to get the physical block number. The second sub-class is the NoJournal class which, as the name implies, is responsible for disabling the journal mapping operations. The main method overrided is _WriteTransactionToLog, so that it doesn't write anything that was supposed to be written to the journal. The NoJournal also forces a transaction flush (ie. forces the block cache to write all the changed blocks) every time the transaction ends, effectively forcing the write to happen as soon as possible. Although this action isn't the most efficient (because the more operations the IO device has queued, the better it can reorganize them into an efficient order) it is there to make sure too much memory won't be used by having too many transactions waiting to be flushed. A better solution would be to only force the flushes when too much memory has been used, but it is also more complex to implement.

The code to handle working with transactions and the journal is based on the code in the BFS implementation because the block cache handles most of the low level operations, and most of the code exists to interface with it, which is mostly the same. The block cache only supports one active transaction at a given moment, which means that when a transaction is started (by the Transaction constructor or the Transaction::Start method), it locks the journal (Journal::Lock), and when the transaction is cancelled (by the Transaction destructor) or finished (by Transaction::Done), the journal is unlocked (Journal::Unlock). In fact, because there is always one active transaction, it is actually managed by the Journal class, instead of the Transaction class, which serves mostly as an interface for simplifying transaction operations inside the former.

The Ext3 journal is laid out as another block device (with the same block size as the file system). The first block contains the journal superblock, and all of the other blocks are for the log. Log blocks can be either log data or log meta-data. They can be distinguished because log meta-data blocks start with a magic number. This means that when the rare occasion of a log data block starting with the magic number happens, we have to remove the magic number and indicate in the block tag that the block has been "escaped". After the magic number, a sequence number follows, so that if the sequence number changes unexpectedly it indicates where the log ends (unless there was some kind of corruption, which is only detectable in an ext4 journal). There are three different types of log meta-data blocks: descriptor blocks, revoke blocks and commit blocks. Descriptor blocks are blocks containing a list of tags. Each tag represents a log data block placed after the descriptor block (the data blocks must be in the same order as their respective tags), and contain the block destination in the file system and some flags associated with the block (like the flag indicating that a block was escaped, and must be restored by placing the magic number at the start before writing it to disk). Revoke blocks contain a list of blocks that aren't supposed to be written back to the file system, in case for example there is a cyclic dependency between two transactions, and one of them hasn't been committed to the journal. However, a block that is marked as revoked in the list inside the revoke block can be written to the file system if its sequence number is higher than the revoke block's sequence number (ie. the block has been changed after it has been revoked, so the new changes must be written). Finally, commit blocks indicate that a transaction has been fully written to the journal, and therefore should be replayed if the disk isn't unmounted correctly.

The journal can have quite a large area reserved for it (for example, about 64MB). This means it can contain a transaction that is larger than what a single descriptor block can represent. I'm not sure how the Linux implementation handles this (it probably limits the maximum transaction size to the maximum size one descriptor block can represent), but this implementation allows for a transaction to grow larger, and split them into smaller transactions (partial transactions) if necessary. Mostly this allows further delaying of a transaction flush, which reduces seek operations and possibly helps with performance. Because the disk still represents each partial transaction as an independent transaction, it is required to make sure all of them are written to disk before writing a commit block, otherwise it can lead the file system into an invalid state. To achieve this, the first commit block is only written after all the other blocks in the transaction. If the disk fails before that first commit block is written, the recovery code detects the missing commit block as an indicator that the first partial transaction (as an independent transaction) hasn't been fully written to disk, and considers it the end of the log (ignoring the other partial transactions that may have been fully written). The _WriteTransactionToLog is responsible for all these operations, and it uses a helper method called _WritePartialTransactionToLog to (as the name implies) write a partial transaction to disk (without the commit block). Supporting larger transactions, while looking like a hack, still provides backward compatibility and can offer some more flexibility. However, more tests are needed to see if a larger transaction support can be an advantage or if it can become a disadvantage.

Most of the journal groundwork has been finished, and has had some simple testing. There are probably some small errors still, which will probably appear when more thorough testing is done. Some things are still missing, like for example separating into the three different modes that the journal handles file system data and meta data (ordered, unordered, or fully journalled). It hasn't been done yet because it requires the journal to be aware of the data operations that happen inside a transaction. Differently from meta-data, data doesn't pass through the block cache, and therefore doesn't easily support transactions. One of the next milestones is file data write support, which will bring more insight into this part. Another thing that hasn't yet been implemented is writing revoke blocks. Currently there is no need for it, and for now it would only offer some minor performance benefits. When it will be required, it will be implemented correctly. One thing that is obvious but still missing is thorough testing. This is very important because from now on, the ext2 code does not force to open a volume read only, and can risk data loss and/or corruption. The journal code didn't take long to implement (probably less than a week), but I had to spend more than a week testing it and fixing some bugs that appeared, including a memory leak inside the block cache. The next blog post will talk about the block allocator implemented for the Ext3 file system, which helped testing the journal code.