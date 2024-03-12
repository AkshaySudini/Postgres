# Custom Vacuum Function in PostgreSQL
the changes were made to `postgres-master/src/backend/commands/vacuum.c`





Changes made to to vacuum.c

## header files
The following header files were added to `vacuum.c`

```c
#include "access/relscan.h"
#include "storage/bufpage.h"

#include "utils/rel.h"
#include "commands/vacuum.h"

```

## function prototypes
The following function prototypes were added to `vacuum.c`
```c
/* Function prototypes */
void custom_vacuum_page(Relation rel, BlockNumber blocknum);
void perform_custom_vacuum(VacuumRelation *vrel);

```

## function implementations
The function implementations were mmade in  `vacuum.c`
```c

/* Function to perform custom vacuum on a single page */
void custom_vacuum_page(Relation rel, BlockNumber blocknum) {
    Buffer buffer;
    Page page;
    OffsetNumber offset, maxOffset;

    /* Read the buffer */
    buffer = ReadBuffer(rel, blocknum);
    LockBuffer(buffer, BUFFER_LOCK_EXCLUSIVE);
    page = BufferGetPage(buffer);

    /* Iterate through page items */
    maxOffset = PageGetMaxOffsetNumber(page);
    for (offset = FirstOffsetNumber; offset <= maxOffset; offset = OffsetNumberNext(offset)) {
        ItemId itemId = PageGetItemId(page, offset);

        if (!ItemIdIsUsed(itemId) || ItemIdIsDead(itemId)) {
            /* Remove dead tuples */
            PageIndexTupleDelete(page, offset);
        }
    }

    /* Optionally, additional logic to compact the page can be added here */

    /* Mark the buffer as dirty and release the lock */
    MarkBufferDirty(buffer);
    UnlockReleaseBuffer(buffer);
}

/* Function to be called for each relation during vacuum */
void perform_custom_vacuum(VacuumRelation *vrel) {
    // Check if vrel is valid
    if (vrel != NULL && vrel->oid != InvalidOid)
    {
        Relation rel = relation_open(vrel->oid, AccessExclusiveLock);

        // Perform custom vacuum operation on the relation
        BlockNumber blocknum, nblocks = RelationGetNumberOfBlocks(rel);
        for (blocknum = 0; blocknum < nblocks; blocknum++) {
            custom_vacuum_page(rel, blocknum);
        }

        // Close the relation
        relation_close(rel, AccessExclusiveLock);
    }
}

```

## function invocation

```c

void
vacuum(List *relations, VacuumParams *params, BufferAccessStrategy bstrategy,
	   MemoryContext vac_context, bool isTopLevel)
{
	static bool in_vacuum = false;

	const char *stmttype;
	volatile bool in_outer_xact,
				use_own_xacts;

	Assert(params != NULL);

	stmttype = (params->options & VACOPT_VACUUM) ? "VACUUM" : "ANALYZE";

	/*
	 * We cannot run VACUUM inside a user transaction block; if we were inside
	 * a transaction, then our commit- and start-transaction-command calls
	 * would not have the intended effect!	There are numerous other subtle
	 * dependencies on this, too.
	 *
	 * ANALYZE (without VACUUM) can run either way.
	 */
	if (params->options & VACOPT_VACUUM)
	{
		PreventInTransactionBlock(isTopLevel, stmttype);
		in_outer_xact = false;
	}
	else
		in_outer_xact = IsInTransactionBlock(isTopLevel);

	/*
	 * Check for and disallow recursive calls.  This could happen when VACUUM
	 * FULL or ANALYZE calls a hostile index expression that itself calls
	 * ANALYZE.
	 */
	if (in_vacuum)
		ereport(ERROR,
				(errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
				 errmsg("%s cannot be executed from VACUUM or ANALYZE",
						stmttype)));

	/*
	 * Build list of relation(s) to process, putting any new data in
	 * vac_context for safekeeping.
	 */
	if (params->options & VACOPT_ONLY_DATABASE_STATS)
	{
		/* We don't process any tables in this case */
		Assert(relations == NIL);
	}
	else if (relations != NIL)
	{
		List	   *newrels = NIL;
		ListCell   *lc;

		foreach(lc, relations)
		{
			VacuumRelation *vrel = lfirst_node(VacuumRelation, lc);
			List	   *sublist;
			MemoryContext old_context;

			sublist = expand_vacuum_rel(vrel, vac_context, params->options);
			old_context = MemoryContextSwitchTo(vac_context);
			newrels = list_concat(newrels, sublist);
			MemoryContextSwitchTo(old_context);
			//change made here -- calling the perform_custom_vacuum function
			perform_custom_vacuum( vrel);
		}
		relations = newrels;
	}
	else
		relations = get_all_vacuum_rels(vac_context, params->options);

	/*
	 * Decide whether we need to start/commit our own transactions.
	 *
	 * For VACUUM (with or without ANALYZE): always do so, so that we can
	 * release locks as soon as possible.  (We could possibly use the outer
	 * transaction for a one-table VACUUM, but handling TOAST tables would be
	 * problematic.)
	 *
	 * For ANALYZE (no VACUUM): if inside a transaction block, we cannot
	 * start/commit our own transactions.  Also, there's no need to do so if
	 * only processing one relation.  For multiple relations when not within a
	 * transaction block, and also in an autovacuum worker, use own
	 * transactions so we can release locks sooner.
	 */
	if (params->options & VACOPT_VACUUM)
		use_own_xacts = true;
	else
	{
		Assert(params->options & VACOPT_ANALYZE);
		if (IsAutoVacuumWorkerProcess())
			use_own_xacts = true;
		else if (in_outer_xact)
			use_own_xacts = false;
		else if (list_length(relations) > 1)
			use_own_xacts = true;
		else
			use_own_xacts = false;
	}

	/*
	 * vacuum_rel expects to be entered with no transaction active; it will
	 * start and commit its own transaction.  But we are called by an SQL
	 * command, and so we are executing inside a transaction already. We
	 * commit the transaction started in PostgresMain() here, and start
	 * another one before exiting to match the commit waiting for us back in
	 * PostgresMain().
	 */
	if (use_own_xacts)
	{
		Assert(!in_outer_xact);

		/* ActiveSnapshot is not set by autovacuum */
		if (ActiveSnapshotSet())
			PopActiveSnapshot();

		/* matches the StartTransaction in PostgresMain() */
		CommitTransactionCommand();
	}

	/* Turn vacuum cost accounting on or off, and set/clear in_vacuum */
	PG_TRY();
	{
		ListCell   *cur;

		in_vacuum = true;
		VacuumFailsafeActive = false;
		VacuumUpdateCosts();
		VacuumCostBalance = 0;
		VacuumPageHit = 0;
		VacuumPageMiss = 0;
		VacuumPageDirty = 0;
		VacuumCostBalanceLocal = 0;
		VacuumSharedCostBalance = NULL;
		VacuumActiveNWorkers = NULL;

		/*
		 * Loop to process each selected relation.
		 */
		foreach(cur, relations)
		{
			VacuumRelation *vrel = lfirst_node(VacuumRelation, cur);

			if (params->options & VACOPT_VACUUM)
			{
				if (!vacuum_rel(vrel->oid, vrel->relation, params, bstrategy))
					continue;
			}

			if (params->options & VACOPT_ANALYZE)
			{
				/*
				 * If using separate xacts, start one for analyze. Otherwise,
				 * we can use the outer transaction.
				 */
				if (use_own_xacts)
				{
					StartTransactionCommand();
					/* functions in indexes may want a snapshot set */
					PushActiveSnapshot(GetTransactionSnapshot());
				}

				analyze_rel(vrel->oid, vrel->relation, params,
							vrel->va_cols, in_outer_xact, bstrategy);

				if (use_own_xacts)
				{
					PopActiveSnapshot();
					CommitTransactionCommand();
				}
				else
				{
					/*
					 * If we're not using separate xacts, better separate the
					 * ANALYZE actions with CCIs.  This avoids trouble if user
					 * says "ANALYZE t, t".
					 */
					CommandCounterIncrement();
				}
			}

			/*
			 * Ensure VacuumFailsafeActive has been reset before vacuuming the
			 * next relation.
			 */
			VacuumFailsafeActive = false;
		}
	}
	PG_FINALLY();
	{
		in_vacuum = false;
		VacuumCostActive = false;
		VacuumFailsafeActive = false;
		VacuumCostBalance = 0;
	}
	PG_END_TRY();

	/*
	 * Finish up processing.
	 */
	if (use_own_xacts)
	{
		/* here, we are not in a transaction */

		/*
		 * This matches the CommitTransaction waiting for us in
		 * PostgresMain().
		 */
		StartTransactionCommand();
	}

	if ((params->options & VACOPT_VACUUM) &&
		!(params->options & VACOPT_SKIP_DATABASE_STATS))
	{
		/*
		 * Update pg_database.datfrozenxid, and truncate pg_xact if possible.
		 */
		vac_update_datfrozenxid();
	}

}

```


## modified files
`vacuum.c`

## installing  the server
 ```bash
 
 ./configure --prefix=/usr/local/pgsql
   make clean && make                                  
   sudo make install

 ```

 starting the server

 ```bash
 sudo mkdir /usr/local/pgsql/data
sudo chown postgres:postgres /usr/local/pgsql/data


sudo -i -u postgres
/usr/local/pgsql/bin/initdb -D /usr/local/pgsql/data


/usr/local/pgsql/bin/pg_ctl -D /usr/local/pgsql/data -l logfile start

 ```