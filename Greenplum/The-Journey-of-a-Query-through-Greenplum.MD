This is a document try to explain the source code of Greenplum database.

# Preface

+ **Objective** 

    The purpose of this document is to explain the source code of Greenplum by following a query processing main flow. This approach try to pickup the critical functions and data structures from the source code, and the function also had been simplified to make it more easier to understand the logic. It serves as a source code reading guide.

    This is not a full design document or full source code analysis. Though the internal and design information of Greenplum is very limited, if you want to get to know more information about that, you can first read the internal document of Postgres in [PostgreSQL Official Document](http://www.postgresql.org/docs/9.5/static/internals.html), or the papers about HAWQ: [HAWQ: a massively parallel processing SQL engine in hadoop.](https://github.com/changleicn/publications/raw/master/hawq-sigmod-2014.pdf).

+ **Target software**

    Greenplum 4.3.6

    Source code in Github:[Greenplum](https://github.com/greenplum-db/gpdb)

    Official website: [Greenplum Site](http://greenplum.org/)
 
+ **License**

    This document is under Creative Commons license BY. 

+ **Author** 
  
    Author: Galy Lee (galylee at gmail dot com)

    Also it belongs to a sub project of [Codeasy](https://github.com/galylee/codeasy/) which is aiming to document the source code of opensource projects, please feel free to contribute to our project.



## Architecture overview 

Greenplum has one **master** and multiple **segments**, actual data is stored in segments, and master accept requests from client and dispatches sub plans to segments, segments get the data from disk and send back to master. Finally master assemble the result and return to client.   


## Master main entry


The main entry of master is in **PostmasterMain()**.

_**Main**_
```C
    src/backend/main/main.c 
    |--Main()
    |--|--PostmasterMain()
```
Like Postgres, Greenplum is using process model. The master listens on its port, it forks a new process to handle the connection request from client.

ServerLoop() is the main idle loop of master.

_**PostmasterMain**_
```C
    src/backend/postmaster/postmaster.c
    /* Postmaster main entry point*/
    |--PostmasterMain()
    |--|--ServerLoop() /*Main idle loop of postmaster*/
         {
             for (;;)
             {
                  if (FD_ISSET(ListenSocket[i], &rmask))  
                  {
                     /*start backend process*/
                     BackendStartup(port);
                   }
              }		
          }
```

When a connection comes, master fork a process in BackendStartup(), and do the client authentication. The new process is usally called as **backend**. Each new connection corresponds to a **backend**.

_**BackendStartup**_
```C
   src/backend/postmaster/postmaster.c
   |--BackendStartup()
     {
         pid = fork_process();
         if (pid == 0)
         {
            /* Perform additional initialization and client authentication */
            BackendInitialize(port);
            /* And run the backend */
            |--BackendRun(port);
            |--|--PostgresMain();
         }
     }
```

## QD：Backend Process

### Backend main entry

After the **backend process** had been forked, it enters a loop in **PostgresMain()** to accept commands or query from client.  When client sends **query** to the backend, backend reads the query out by **ReadCommand()**, and dispatchs it to proper function according to its **first char**.The meaning of the first char is defined by the **frontend/backend protocol**. For example 'Q' means the comand is a simple query from the frontend,  it is routed to **exec_simple_query()** for handling. 

_**PostgresMain**_
```C
    src/backend/tcop/postgres.c
    /* postgres main loop 
     * all backends, interactive or otherwise start here
     */
    |--PostgresMain()
       {
        for (;;)
        {	
            /*
             * (3) read a command (loop blocks here)
             */
              firstchar = ReadCommand(&input_message);
			
            /*
             * (6) process the command.  But ignore it if we're skipping till
             * Sync.
             */
              switch (firstchar)
             {
                case 'Q':   /* simple query */
                {
                   exec_simple_query(query_string, NULL, -1); 
                }
                case 'M':    /* MPP dispatched stmt from QD */
                case 'T':    /* MPP dispatched dtx protocol command from QD */
                case 'P':    /* parse */
                case 'W':    /* GPDB QE-backend startup info (motion listener, version). */
                case 'B':    /* bind */
                case 'E':    /* execute */
                case 'F':    /* fastpath function call */
                case 'C':    /* close */
                case 'D':    /* describe */
                case 'H':    /* flush */
                case 'S':    /* sync */
                case 'X':
                case EOF:
                case 'd':    /* copy data */
                case 'c':    /* copy done */
                case 'f':    /* copy fail */
             }
         }
      }
```
Comparing with Postgres, Greenplum had added three messages, all of them are relative to master/segment communication.
```C
                case 'M':    /* MPP dispatched stmt from QD */
                case 'T':    /* MPP dispatched dtx protocol command from QD */
                case 'W':    /* GPDB QE-backend startup info (motion listener, version). */
```


### QD query processing

**exec_simple_query** do most of the query processing work. As you know, the query is in SQL standard format, the backend need to parse, analyze, rewrite, and plan the query, and do the execution according the plan, finally send the result tuples back to the client.

_**exec_simple_query**_
```C
    src/backend/tcop/postgres.c 
    |--exec_simple_query()
      {
        /*
         * Start up a transaction command.
         */
         start_xact_command();

         /*
         * Do basic parsing of the query or queries 
         */
         parsetree_list = pg_parse_query(query_string);	
	
         /*
          * OK to analyze, rewrite, and plan this query.
          */	
         querytree_list = pg_analyze_and_rewrite(parsetree, query_string,NULL, 0);

         plantree_list = pg_plan_queries(querytree_list, NULL, false);	

         /*
         * Create unnamed portal to run the query or queries in.
         */
         portal = CreatePortal("", true, true);	
	
         /* 
          * Start the portal. 
          */
         PortalStart();
					
        /*
         * Now we can create the destination receiver object.
         */
         receiver = CreateDestReceiver(dest, portal);

         /*
          * Run the portal to completion, and then drop it (and the receiver).
          */
         (void) PortalRun();

         (*receiver->rDestroy) (receiver);
	
         PortalDrop(portal, false);
	
         /*
         * Close down transaction statement, if one is open.
         */
         finish_xact_command();
      	
      }	
```


#### Query Parsing
The query parsing is done by 
 `parsetree_list = pg_parse_query(query_string);`
Given a query in string form, it do lexical and grammatical analysis, and return a parse tree.

Postgres uses Flex/Bison as program interptreter to parse SQL query or command.

The task of discovering the source structure again is decomposed into subtasks: 
 1. Split the source file into tokens (Flex). 
 2. Find the hierarchical structure of the program (Bison).


_**Flex**_

flex is a tool for generating scanners: programs which recognized lexical patterns in text. flex reads the given input files, or its standard input if no file names are given, for a description of a scanner to generate.

For example, in the following code, 'digit' is defined as the regular expression [0-9], token like 8989 is reconigzed as digit.

```C
   gpdb/src/backend/parser/scan.l
   ....
   whitespace		({space}+|{comment})

   digit		[0-9]
   ident_start		[A-Za-z\200-\377_]
   ident_cont		[A-Za-z\200-\377_0-9\$]
   ....
```

_**Bison**_
Bison is a general-purpose parser generator.

For example, a simple select has three parameters as following
```SQL   
    SELECT *                 /* target_list */
    FROM user                /* from_clause */
    Where username= 'JOHN' ' /* where_clause */
```
A simple select statment is define in the following code. The Bison reconigzes the above statment and its parameters, and create a **SelectStmt** data structure, then fills the parameters in it.
```C
    src/backend/parser/gram.y 
    simple_select: 
          SELECT opt_distinct target_list 
                 into_clause from_clause where_clause 
                 group_clause having_clause window_clause 
           { 
                 SelectStmt *n = makeNode(SelectStmt); 
                 n->distinctClause = $2; 
                 n->targetList = $3; 
                 n->intoClause = $4; 
                 n->fromClause = $5; 
                 n->whereClause = $6; 
                 n->groupClause = $7; 
                 n->havingClause = $8; 
                 n->windowClause = $9; 
                 $$ = (Node *)n; 
            } 
```
The structure to hold the parse result is **SelectStmt**, all of the options in SELECT statment are defined. 

_**SelectStmt**_
```C
src/include/nodes/ parsenodes.h
typedef struct SelectStmt 
{ 
 	List	   *distinctClause;  
 	IntoClause *intoClause;		/* target for SELECT INTO / CREATE TABLE AS */ 
 	List	   *targetList;		/* the target list (of ResTarget) */ 
 	List	   *fromClause;		/* the FROM clause */ 
 	Node	   *whereClause;	/* WHERE qualification */ 
 	List	   *groupClause;	/* GROUP BY clauses */ 
 	Node	   *havingClause;	/* HAVING conditional-expression */ 
 	List	   *windowClause;	/* window specification clauses */ 
 	List       *scatterClause;	/* GPDB: TableValueExpr data distribution */ 
 	WithClause *withClause; 	/* with clause */ 
 
        ........
 
	/* This field used by: SELECT INTO, CTAS */ 
 	List       *distributedBy;  /* GPDB: columns to distribute the data on. */ 
 
 
 } SelectStmt; 
```

_**Analyze and Rewrite**_

After the query is parsed, it needs to transform into database internal object ID, for example given a table name, it need to refer to system meta data to get the table id. **parse_analyze** performs parse analysis.

Also rewrite the queries as necessary, in the rewrite, it fires the rules.

```C
|--pg_analyze_and_rewrite()
   {  
      /*(1) Perform parse analysis.*/
      |--querytree_list = parse_analyze(parsetree, query_string, paramTypes, numParams); 
      |--|--do_parse_analyze(parseTree, pstate);
      |--|--|--query = transformStmt(pstate, parseTree, &extras_before, &extras_after);
      |--|--|--|--result = transformSelectStmt(pstate, n);

      /* (2) Rewrite the queries, as necessary*/
      |--querytree_list = pg_rewrite_queries(querytree_list);
      |--|--QueryRewrite()
      |--|--|-- /*Apply all non-SELECT rules possibly getting 0 or many queries*/
      |--|--|--querylist = RewriteQuery(parsetree, NIL);
      |--|--|--|-- fireRules(parsetree,)
      |--|--|--|-- /* Apply all the RIR rules on each query */ 
      |--|--|--|-- query = fireRIRrules(query, NIL);

    }
```

**transformStmt**
In the transformStmt(), `rte->relid = RelationGetRelid(rel);`, the relation id of source table is retrieved. 
```C
   src/backend/parser/analyze.c
   |--transformStmt
   |--|--transformFromClause(pstate, stmt->fromClause);
   |--|--|--transformFromClauseItem(pstate,
   |--|--|--|--transformTableEntry(pstate, rangeVar);
   |--|--|--|--|--addRangeTableEntry(pstate, r, r->alias
                  {
                     RangeTblEntry *rte = makeNode(RangeTblEntry);
                     char	   *refname = RelationGetRelationName(rel);
                     rte->relid = RelationGetRelid(rel);
                     ......
                   }
```
**Query struct**

Query strcture hold the result of transformed parsing tree.

```C
typedef struct Query 
{ 
    NodeTag		type; 
    CmdType		commandType;	/* select|insert|update|delete|utility */ 
    QuerySource querySource;	/* where did I come from? */ 
    bool		canSetTag;		/* do I set the command result tag? */ 
    Node	   *utilityStmt;	/* non-null if this is DECLARE CURSOR or a 
    int			resultRelation; /* rtable index of target relation for 
    IntoClause *intoClause;		/* target for SELECT INTO / CREATE TABLE AS */ 
    bool		hasAggs;		/* has aggregates in tlist or havingQual */ 
    bool		hasWindFuncs;	/* has window function(s) in target list */ 
    bool		hasSubLinks;	/* has subquery SubLink */  
    List	   *rtable;			/* list of range table entries */ 
    FromExpr   *jointree;		/* table join tree (FROM and WHERE clauses) */ 
    List	   *targetList;		/* target list (of TargetEntry) */  
    List	   *returningList;	/* return-values list (of TargetEntry) */ 
    .......
}
```

#### Planning
```
For example:

    SELECT  *
    FROM    tab1, tab2, tab3, tab4
    WHERE   tab1.col = tab2.col AND
        tab2.col = tab3.col AND
        tab3.col = tab4.col

    Tables 1, 2, 3, and 4 are joined as:
    {1 2},{2 3},{3 4}
    {1 2 3},{2 3 4}
    {1 2 3 4}
    (other possibilities will be excluded for lack of join clauses)
```

The primary entry point is planner().
```C
|--grouping_planner()
  {
     /* Generate the best unsorted and presorted paths for this Query */
     |--query_planner()
     |--|--make_one_rel 
          /*Finds all possible access paths for executing a query*/ 
          { 
              /* find seqscan and all index paths for each base relation */
              set_base_rel_pathlist() 

             /* Generate access paths for the entire join tree.*/
              make_rel_from_joinlist()
           }
  
     /* reate a plan according to query_planner's * results.*/
     result_plan = create_plan(root, best_path);
  
     /* Insert AGG or GROUP node if needed */
     make_agg()
  
     /*If there is a DISTINCT clause, add the UNIQUE node.*/
     make_unique()
  
     /*Finally, if there is a LIMIT/OFFSET clause, add the LIMIT node.*/
     make_limit()
  } 
```

**Relation Access Paths**

set_base_rel_pathlist() finds seqscan and all index paths for each base relation. It considers sequential scan, index and bitmap scans, TID scan for a base relation.

```C
      |--set_base_rel_pathlist()
         {
            for (rti = 1; rti < root->simple_rel_array_size; rti++)
            {
                |--set_rel_pathlist(root, rel, rti);
                |--|--set_plain_rel_pathlist(root, rel, rte);
                      {
                         /* Consider sequential scan. */
                         seqpath = create_seqscan_path(root, rel);

                         /* Consider index and bitmap scans */
                         create_index_paths(root, rel, );

                         /* Consider TID scans */
                         create_tidscan_paths(root, rel, &tidpathlist);

                         /* Now find the cheapest of the paths for this rel */
	                 set_cheapest(root, rel);
                       }
             }
```

**seqscan_path**

It makes a T_SeqScan type node, and also it caculates the cost for the sequential scan in `cost_seqscan(pathnode, root, rel)` . 

```C
Path *
create_seqscan_path(PlannerInfo *root, RelOptInfo *rel)
{
        Path	   *pathnode = makeNode(Path);

        pathnode->pathtype = T_SeqScan;
 
        pathnode->locus = cdbpathlocus_from_baserel(root, rel);
 
        cost_seqscan(pathnode, root, rel);

        return pathnode;
}
```

**find an optimal way to join all relations into one relation**

We employ a simple "dynamic programming" algorithm: we first find all ways to build joins of two jointree items, then all ways to build joins of three items (from two-item joins and single items), then four-item joins, and so on until we have considered all ways to join all the items into one rel.

```C
        |--make_rel_from_joinlist()
           Generate access paths for the entire join tree.
        |--|--make_rel_from_joinlist
        |--|--|--make_one_rel_by_joins
                 {
                    /*
                     * We employ a simple "dynamic programming" algorithm
                     */
                     for (lev = 2; lev <= levels_needed; lev++)
                     {
                        |--make_rels_by_joins
                        {
                            /* First, consider left-sided and right-sided plans */
                            |--make_join_rel
                            |--|--add_paths_to_joinrel
                                  {
                                      /*
	                               * 1. Consider mergejoin paths where both relations 
	                               *  must be explicitly sorted.
	                               */
	                               |--sort_inner_and_outer();
                                       |--|--create_mergejoin_path()

                                       match_unsorted_outer();
                                       match_unsorted_inner();
                                       hash_inner_and_outer();
                                   }
                                     
                           /* Now, consider "bushy plans" */
                       }
                 }
```

**create_mergejoin_path**

In merge join, it adds motion nodes above subpaths and decide where to join.

```C
    |--create_mergejoin_path()
       {
           |--cdbpath_motion_for_join()
             {
                /*
                 * Add motion nodes above subpaths and decide where to join.
                 */
                 |--cdbpath_create_motion_path()
                 |--|--cdbpath_cost_motion(root, pathnode);
                 
                 pathnode = makeNode(MergePath);
                 
                 cost_mergejoin(pathnode, root);
              }
         }

```

#### Execution
**_PortalStart_**
```C
    |--PortalStart
    |--|--ExecutorStart(queryDesc, eflags);	
         {
               /*
                * Initialize the plan state tree
                */
                InitPlan(queryDesc, eflags);
			
                if (Gp_role == GP_ROLE_DISPATCH && dispatch == DISPATCH_PARALLEL)
                {
                        /* Assign gang descriptions to the root slices of the slice forest. */
                        InitRootSlices(queryDesc);
				
                        /*
                         * Since we intend to execute the plan, inventory the slice tree,
                         * allocate gangs, and associate them with slices.
                         */
                         |--AssignGangs(queryDesc, gp_singleton_segindex);
                         |--|--allocateGang()
                         |--|--|--createGang()
                         |--|--|--|--gp_pthread_create(&pParms->thread, thread_DoConnect)
                         |--|--|--|--|--thread_DoConnect()	
                         |--|--|--|--|--|--cdbconn_doConnect()	
                         |--|--|--|--|--|--|--PQconnectdbParams()
                         |--|--|--|--|--|--|--PQsetNoticeReceiver()

                         /*			
                          * if in dispatch mode, time to serialize plan and query
                          * trees, and fire off cdb_exec command to each of the qexecs
                          */				
                         |--cdbdisp_dispatchPlan();
                         |--|--cdbdisp_dispatchX()
                         |--|--|--cdbdisp_dispatchToGang(); 
                  }
           }	
```		

# 2. Segment

## QE：Backend Process

### PostgresMain
__**BackendRun**__
```C
BackendRun
    PostgresMain
        for (;;)
		{	
			switch (firstchar)
			{
				case 'M':           /* MPP dispatched stmt from QD */
				{
						exec_mpp_query();		
				}
				.......
			}
		}  

```

### Query Processing
_**exec_mpp_query**_

When QE receives the plan dispatched from QD, it calls exec_mpp_query to handle it. Normally, a query need to go through the process of parse/analysis/rewrite/plan/execution/return result, but because the query had already been planned at QD, so it deserialize the query execution plan from QD, and excutes it, finnally it returns the result back to QD.
 
```C
/*
 * exec_mpp_query
 *
 * Called in a qExec process to read and execute a query plan sent by
 * cdbdisp_dispatchPlan().
 *
 * Caller may supply either a Query (representing utility command) or
 * a PlannedStmt (representing a planned DML command), but not both.
 */
exec_mpp_query
{
    Assert(Gp_role == GP_ROLE_EXECUTE);

    start_xact_command();
	
    /*
     * Deserialize the slice table, if there is one, and set up the local slice.
     */
    sliceTable = (SliceTable *) deserializeNode(serializedSliceInfo);
	
    /*
     * Deserialize the query execution plan (a PlannedStmt node), if there is one.
     */    
    plan = (PlannedStmt *) deserializeNode(serializedPlantree,serializedPlantreelen);
	
    /*
     * Create unnamed portal to run the query or queries in. If there
     * already is one, silently drop it.
     */
    portal = CreatePortal("", true, true);
	
    /* 
    * Start the portal.  No parameters here.
    */
    PortalStart(portal, paramLI, InvalidSnapshot,seqServerHost, seqServerPort);

    /*
    * Now we can create the destination receiver object.
    */
    receiver = CreateDestReceiver(dest, portal);					

    /*
     * Run the portal to completion, and then drop it (and the receiver).
     */
    (void) PortalRun();
		
    (*receiver->rDestroy) (receiver);

    PortalDrop(portal, false);

    finish_xact_command();
		
}
```

### Initialization
_**PortalStart**_

Before the execution, the executor needs to do some initialization work. One of the noticable work is to setup the inter connect between segements for data transfer between them. 

```C
PortalStart
	ExecutorStart(queryDesc, eflags);	
	{	
		if (Gp_role == GP_ROLE_EXECUTE)
		{
			for (i=1; i <= queryDesc->plannedstmt->nMotionNodes; i++)
			{
				InitMotionLayerNode(estate->motionlayer_context, i);
			}
			
			SetupInterconnect(estate);
		}		
		/*
		 * Initialize the plan state tree
		 */
		InitPlan(queryDesc, eflags);
	}

```
EState is used to store the information about execution. The slices table and interconnection information are stored here.

```C
typedef struct EState
{
         ......
	/* Additions for MPP plan slicing. */
	struct SliceTable *es_sliceTable;

	void	   *motionlayer_context;  /* Motion Layer state */
	struct ChunkTransportState *interconnect_context; /* Interconnect state */
        ......

}
```
A distributed plan is divided into slices, a slice is an indepente executable portion of plan, each motion node create two slices. All of the slice in a plan are store in a slice table. Each slice has its index number and gang, which is the excuting nodes of the slice

```C
typedef struct Slice
{
	NodeTag type;

	/*
	 * The index in the global slice table of this
	 * slice.  The root slice of the main plan is
	 * always 0. Slices that have senders at their
	 * local root have a sliceIndex equal to the
	 * motionID of their sender Motion.
	 *
	 * Undefined slices should have this set to
	 * -1.
	 */
	int sliceIndex;

	/*
	 * The root slice of the slice tree of which
	 * this slice is a part.
	 */
	int rootIndex;

	/*
	 * the index of parent in global slice table (origin 0)
	 * or -1 if this is root slice.
	 */
	int parentIndex;

	/*
	 * An integer list of indices in the global slice
	 * table (origin  0) of the child slices of
	 * this slice, or -1 if this is a leaf slice.
	 * A child slice corresponds to a receiving
	 * motion in this slice.
	 */
	List *children;

	struct Gang *primaryGang;

	/*
	 * A list of CDBProcess nodes corresponding to
	 * the worker processes allocated to implement
	 * this plan slice.
	 *
          */
	List *primaryProcesses;
} Slice;
```

```C
typedef struct MotionNodeEntry
{
	/*
	 * First value in entry has to be the key value.  The key is the motion
	 * node ID.
	 */
	int16           motion_node_id;


	/*
	 * Our route-based array of htup_fifos, for the case where we are a merge receive.
	 */
	ChunkSorterEntry *ready_tuple_lists;

	/* The description of tuples that this motion node will be exchanging. */
	TupleDesc       tuple_desc;

	/*
	 * If preserve_order is false, this is used to hold completed tuples that
	 * have not yet been consumed.  If preserve_order is true, this is NULL.
	 */
	htup_fifo       ready_tuples;
}
```

The connections of between motion peers are store in the ChunkTransportStateEntry.
```C
typedef struct ChunkTransportStateEntry
{

    /* Connection array: first the primaries, then the mirrors (if needed) */
    MotionConn *conns;

    /* slice table entries */
    struct Slice   *sendSlice;
    struct Slice   *recvSlice;

} ChunkTransportStateEntry;

```

_**SetupTCPInterconnect**_

Before the exeuction, the interconnect needs to setup the outoing and incoming connections. Greenplum supports both TCP and UDP connections.

```C
      |--SetupTCPInterconnect()
         {
            /* Initiate outgoing connections. */
            sendingChunkTransportState = startOutgoingConnections();

	   /* now we'll do some setup for each of our Receiving Motion Nodes. */
           foreach(cell, mySlice->children)
           {
                 aSlice = (Slice *) list_nth();

                 |--createChunkTransportState();
                    {
                        pEntry = &transportStates->states[motNodeID - 1];

                         	for (i = 0; i < pEntry->numConns; i++)
                         {
                            MotionConn *conn = &pEntry->conns[i];

                            /* Initialize MotionConn entry. */
                            conn->state = mcsNull;
                            conn->sockfd = -1;
                          }
                    }
           }
        }

```

### Query Execution
_**PortalRun**_

The actual execution of the plan is in PortalRun, a simple query is dispatched to ExecutePlan, but the DDL commands is dispatched to PortalRunUtility. 

```C		
PortalRun
   switch (portal->strategy)
   {
      case PORTAL_ONE_SELECT:
            |--(void) PortalRunSelect(portal, true, count, dest);
            |--|--ExecutorRun()
            |--|--|--ExecutePlan()
      case PORTAL_MULTI_QUERY:
             /* Execute multi queries or non-SELECT-like queries*/
            |--PortalRunMulti(portal, isTopLevel, dest, altdest, completionTag);
            |--|--PortalRunUtility(portal, stmt, isTopLevel, altdest, NULL);
                  /* process utility functions (create, destroy, etc..) */
   }
				   
```

_**ExecutePlan**_

Postgres Executor uses a demand-pull pipeline mechanism to process the whole plan tree. ExecProcNode pull one tuple from the top node in the plan, which recruisively execute its child nodes in the plan. 

```C
   ExecutePlan()	
   {
         /*
          * Process BEFORE EACH STATEMENT triggers
          */
         .......		
         
         /*
          * Loop until we've processed the proper number of tuples from the plan.
          */
         for (;;)
         {
              planSlot = ExecProcNode(planstate);
   
              /*
              * Based on the operation, a tuple is either              
              * returned it to the user (SELECT) or inserted, deleted, or updated.
              */
              switch (operation)
              {
                    case CMD_SELECT:
        	             ExecSelect(slot, dest, estate);
        	             result = slot;
                    break;
                    .......
              }
          }

         /*
          * Process AFTER EACH STATEMENT triggers
          */
         ......

    }
```
ExecProcNode call different node handler according to its node type. 

```C
     /* Execute the given node to return a(nother) tuple. */
     |--ExecProcNode()
        {
	   switch (nodeTag(node))
	   {
                 /*
                  * scan nodes
                  */
                  case T_IndexScanState:
                         result = ExecIndexScan((IndexScanState *) node);
                         break;
                 /*
                  * join nodes
                  */
                 case T_NestLoopState:
                         result = ExecNestLoop((NestLoopState *) node);
                         break;
                 ....

                 /*
                 * shareinput nodes
                 */
                 case T_ShareInputScanState:
                         result = ExecShareInputScan((ShareInputScanState *) node);
                         break;
                 /*
                  * materialization nodes
                  */
                 case T_SortState:
                         result = ExecSort((SortState *) node);
                         break;
                 case T_MotionState:
                         result = ExecMotion((MotionState *) node);
                         break;
                 ....
           }
        }
```
For a NestLoop, it gets one tuple from outer node and then get another tuple from inner node, if two tuple satisfies qualification, it returns the joinned tuple. 

```C
   src/backend/executor/nodeNestloop.c
   |--ExecNestLoop()
     {
	/*
	 * Ok, everything is setup for the join so now loop until we return a
	 * qualifying join tuple.
	 */
	
	for (;;)
	{
                /*
                 * If we don't have an outer tuple, get the next one and reset the
                 * inner scan.
                 */
                  outerTupleSlot = ExecProcNode(outerPlan);

                /*
                 * we have an outerTuple, try to get the next inner tuple.
                 */
                 innerTupleSlot = ExecProcNode(innerPlan);
                
                /*
                 * at this point we have a new pair of inner and outer tuples so we
                 * test the inner and outer tuples to see if they satisfy the node's
                 * qualification.                 
                 *
                 */

                 if (ExecQual(joinqual, econtext, node->nl_qualResultForNull))
                 {

                     /*
                      * qualification was satisfied so we project and return the
                      * slot containing the result tuple using ExecProject().
                      */
                      return ExecProject(node->js.ps.ps_ProjInfo, NULL);
		}
          }
      }
```
For a scan node, it calls the access method according its access method type.

```C

   /*
    *   Scans the relation using the 'access method' indicated and
    *	returns the next qualifying tuple
    */

   |--ExecScan()
      {
          /*
           * get a tuple from the access method loop until we obtain a tuple which
           * passes the qualification.
           */
           for (;;)
           {
                slot = (*accessMtd) (node);
                /*
                 * check that the current tuple satisfies the qual-clause
                 */
                if (!qual || ExecQual(qual, econtext, false))
                {
                      /*
                       * Found a satisfactory scan tuple.
                       */
                       return slot;
                }
            }
       }

```
_**ExecMotion**_
```C
   src/backend/executor/nodeMotion.c
   |--ExecMotion()
      {
           /*
            * at the top here we basically decide: -- SENDER vs. RECEIVER and --
            * SORTED vs. UNSORTED
            */
           if (node->mstype == MOTIONSTATE_RECV)
           {
               if (motion->sendSorted)
               {
                  if (gp_enable_motion_mk_sort)
                      tuple = execMotionSortedReceiver_mk(node);
                  else
                      tuple = execMotionSortedReceiver(node);
               }  else
                      tuple = execMotionUnsortedReceiver(node);
           } 
           else if(node->mstype == MOTIONSTATE_SEND)
           {
                return execMotionSender(node);
           }
       }
```

```C
    src/backend/executor/nodeMotion.c
    |--execMotionUnsortedReceiver()
    |--|--RecvTupleFrom()
          {
             pMNEntry = getMotionNodeEntry(mlStates, motNodeID, "RecvTupleFrom");

             processIncomingChunks();

             *tup_i = htfifo_gettuple(pMNEntry->ready_tuples);
          }
```
### Send the result back

_**CreateDestReceiver**_
```C						
CreateDestReceiver
	switch (dest)
	{
		case DestRemote:
		case DestRemoteExecute:
			return printtup_create_DR(dest, portal);	
	}
```

_**printtup_create_DR**_
```C	
printtup_create_DR
	self->pub.receiveSlot = printtup_20;
	self->pub.rStartup = printtup_startup;
	self->pub.rShutdown = printtup_shutdown;
	self->pub.rDestroy = printtup_destroy;	
```

_**printtup_startup**_
```C	
printtup_startup
	pq_puttextmessage('P', portalName);
	SendRowDescriptionMessage();
```
_**printtup_20**_
```C	
printtup_20
	pq_beginmessage(&buf, 'D');
	pq_sendcountedtext(&buf, outputstr, strlen(outputstr), true);
	pq_endmessage(&buf);

