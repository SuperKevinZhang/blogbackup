---
title: Housekeeping设计
date: 2017-10-18 12:37:06
tags: [数据库,housekeeping]
---

**分享一下自己做的一个Housekeeping案例**

​	项目一般运行一段时间后,就会产生一些历史数据,这些数据虽然有用,但是平时我们用不到(比如快递公司的运单数据,一般很少会查询1-2年前的运单),为了节省宝贵的服务器资源,这个时候我们就需要把历史数据迁移到其他库或者机器上去;这个过程就是Housekeeping技术;

<!--more-->

​	做Housekeeping需要从以下几个方面去考虑

 *	清理范围


 *  清理条件

 *  清理周期

    ​

    **清理范围** 整理如下表格

| 主表数据          | 说明   | CWB  查询 | CWB  明细查询 | AWB查询  及其明细 | AWB  Check |
| ------------- | ---- | ------- | --------- | ----------- | ---------- |
|               |      |         |           |             |            |
| TB_CWB        |      | ●       | ●         | ●           | ●          |
| TB_BWB        |      |         |           | ●           |            |
| TB_AWB        |      |         | ●         | ●           |            |
| TB_CWBWEIGHT  |      | ●       | ●         | ●           | ●          |
| TB_CWBXDWPATH |      | ●       |           |             |            |
|               |      |         |           |             |            |




​	**清理条件**

​		一般根据日期来决定,根据业务不同,清理时间不同

​	**清理周期**

​		如没一周执行一次清理记录,根据清理条件把之前的数据都清理掉

​		**注意:很多人会忘记,清理结束后,记得重建相关索引**



附上相关代码:

Housekeeping的相关存储过程

``` sql
SET QUOTED_IDENTIFIER OFF 
GO
SET ANSI_NULLS ON 
GO


ALTER  PROCEDURE sp_ICLSKHistClearing
AS

BEGIN
	DECLARE @AppName 	varchar(30),
		@MsgText 	varchar(512),
		@rows 		int 

	DECLARE @DebugFlag	smallint	-- Debug Flag: 0=Disabled;1=Enabled

	DECLARE	@rowcount 	int,		-- @@ROWCOUNT value.
		@error 		int		-- @@ERROR value.

	DECLARE	@table_name 	varchar(30),	-- DEL_DATA_CONFIG.TABLE_NAME
		@del_data_type 	char,		-- DEL_DATA_CONFIG.DEL_DATA_TYPE
		@day_of_week	varchar(7),	-- DEL_DATA_CONFIG.DAY_OF_WEEK
						-- (1=Sunday,2=Monday,...,7=Saturday)
		@column_name 	varchar(30),	-- DEL_DATA_CONFIG.COLUMN_NAME
		@keep_days 	smallint,	-- DEL_DATA_CONFIG.KEEP_DAYS
		@tbl_cd 	smallint,	-- DEL_DATA_CONFIG.TBL_CD (Period Type)
						-- (Day=7;Week=8;Month=11;Year=15)
		@stat		smallint,	-- DEL_DATA_TYPE.STAT
		@sql_rule	varchar(1024),	-- DEL_DATA_CONFIG.SQL_RULE
		@sp_flag	smallint	-- DEL_DATA_CONFIG.SP_FG

	DECLARE @Timestamp 	datetime,	-- Retention date.
		@TargetDate 	varchar(32)	-- Retention date ("YYYYMMDD").

	DECLARE @sql 		varchar(1024)	-- Close to limit 1024 max,
						-- don't change SQL text without checking.
	DECLARE @sql2		varchar(1024)	-- Utility buffer

	DECLARE @SPPurgeName	varchar(50)	-- Stored Procedure for purging.
	DECLARE @rc		int		-- Return Code of the Stored Procedure.

	/****************************************************************/
	/* Initialise.							*/
	/****************************************************************/

	set NOCOUNT on		-- Turns off the message returned at the end of each statement.
	set ROWCOUNT 2000	-- To avoid using too many locks in large delete operations.

-----------------------------
	select @DebugFlag = 0	-- Debug functionality: 0=Disabled;1=Enabled 
-----------------------------

	select @rc = 0		-- Clear the Stored procedure return code.

	select @AppName = 'sp_ICLSKHistClearing'

	select @MsgText = @AppName + ' started'
	exec sp_LogMsg 50010, @MsgText, Informational

	/****************************************************************/
	/* Main loop.							*/
	/****************************************************************/

	DECLARE	del_cursor CURSOR FOR
		select	TABLE_NAME,
			Upper(DEL_DATA_TYPE),
			DAY_OF_WEEK,
			COLUMN_NAME,
			IsNull(KEEP_DAYS, 0),
			TBL_CD,
			STAT,
			IsNull(SQL_RULE, ''),
			SP_FG
		from	DEL_DATA_CONFIG
		order by GROUP_ID, SEQ_ID

	OPEN del_cursor

	FETCH NEXT FROM del_cursor INTO @table_name, @del_data_type, @day_of_week, @column_name, @keep_days, @tbl_cd, @stat, @sql_rule, @sp_flag

	while (@@FETCH_STATUS <> -1)
	begin
		if (@@FETCH_STATUS = -2)
		begin
			FETCH NEXT FROM del_cursor INTO @table_name, @del_data_type, @day_of_week, @column_name, @keep_days, @tbl_cd, @stat, @sql_rule, @sp_flag
			continue
		end

		/****************************************************************/
		/* Check if the current row must be processed today.		*/
		/****************************************************************/

		if (select charindex(convert(varchar(1), datepart(dw, getdate())), @day_of_week)) = 0
		begin
			FETCH NEXT FROM del_cursor INTO @table_name, @del_data_type, @day_of_week, @column_name, @keep_days, @tbl_cd, @stat, @sql_rule, @sp_flag
			continue
		end

		/****************************************************************/
		/* Initialise local counts and variables.			*/
		/****************************************************************/

		select @rows = 0	-- Accumulates count of target rows
					-- deleted for each row of DEL_DATA_CONFIG.

		select @Timestamp = dateadd(day, -@keep_days, getdate())
		select @TargetDate = convert(varchar, @Timestamp, 112)
		select @TargetDate = 'convert(datetime, ''' + @TargetDate + ''')'

		select @sql = '', @sql2 = ''

		/****************************************************************/
		/*								*/
		/*	Run "sp_PURGE_xxxxxxxx" Stored Procedure.		*/
		/*								*/
		/****************************************************************/

                if @sp_flag = 1
                	begin
		        	select @SPPurgeName = 'sp_PURGE_' + @table_name

				if @DebugFlag = 0
				begin
					if @Stat is null 
						begin
			        			exec @rc = @SPPurgeName @Timestamp
						end
					else
						begin
							exec @rc = @SPPurgeName @Timestamp, @Stat
						end
					while @rc > 0
					begin
						select @rows = @rows + @rc
						if @Stat is null 
							begin
			        				exec @rc = @SPPurgeName @Timestamp
							end
						else
							begin
								exec @rc = @SPPurgeName @Timestamp, @Stat
							end
					end
				end
				else select @SPPurgeName

				if @rc < 0
				begin
					select @MsgText = @AppName + ': Error executing ''%1'' Stored Procedure. RC=' + convert(varchar(12), @rc)
					exec sp_LogMsg 50001, @MsgText, Error, @SPPurgeName
					break
				end
				else if @rc = 0
				begin
					select @MsgText = @table_name + ' Cleaned successfully'
--CarloC				exec sp_LogMsg 50012, @MsgText, Informational
					FETCH NEXT FROM del_cursor INTO @table_name, @del_data_type, @day_of_week, @column_name, @keep_days, @tbl_cd, @stat, @sql_rule, @sp_flag
					continue
				end
			end

		/****************************************************************/
		/*								*/
		/*	Prepare and run a Delete statement.			*/
		/*								*/
		/****************************************************************/

		/****************************************************************/
		/* General: Clear general tables, including the following:	*/
		/*	TRANSFER	  TRANS_HEADER	  INVENTORY_EVENT	*/
		/*	PO					  		*/
		/****************************************************************/

		if @del_data_type = 'G'
		begin
			if @column_name = '*' and @keep_days = 0
			begin
				-- Delete From <config.table_name>
				select @sql = 'Delete From '
				 + @table_name
				  + ' Where 1=1'
			end
			else if @stat Is Null
			begin
				-- Delete From <config.table_name> 
				-- Where  <config.column_name> has date < today's date - <config.keep_days>
				select @sql = 'Delete From '
				 + @table_name
				  + ' Where '
				   + @column_name
				    + '<' + '' + @TargetDate + ''
			end
			else
			begin
				-- Delete From <config.table_name> 
				-- Where  <config.table_name>.STAT = <config.stat>  
				-- And	  <config.column_name> has date < today's date - <config.keep_days>
				select @sql = 'Delete From '
				 + @table_name
				  + ' Where STAT='
				   + convert(varchar(10), @stat)
				    + ' And '
				     + @column_name
				      + '<' + '' + @TargetDate + ''
			end
		end

		/****************************************************************/
		/* Transaction: Clear transaction tables through TRANS_HEADER.	*/
		/*	APPROVAL	BINARY_TLOG	GIFT_CERT_TENDER	*/
		/*	GL_TRAN		LN_DETAIL	LN_DETAIL_SRLZD		*/
		/*	LN_DISCOUNT	LN_LINK		LOAN_PICKUP		*/
		/*	MANUAL_PRICE	PAYMENT		PLU_NOT_FOUND		*/
		/*	RETURN_TRAN 	TAX_MOD_INFO 	TENDER_AUTHRZATION	*/
		/*	TENDER_DETAIL	TRANSACTION_TAX	TRANSACTION_TENDER	*/
		/*	TRANSACTION_TEXT					*/
		/****************************************************************/

		else if @del_data_type = 'T'
		begin
			-- Delete From <config.table_name> From TRANS_HEADER
			-- Where  <config.table_name>.TRAN_ID = TRANS_HEADER.TRAN_ID
			-- And    TRANS_HEADER.TRAN_STRT_TS has a date less than (today's date - <config.keep_days>)

			select @sql = 'Delete From '
		 	 + @table_name
		  	  + ' From TRANS_HEADER Where ' 
			   + @table_name
			    + '.TRAN_ID = TRANS_HEADER.TRAN_ID And TRANS_HEADER.TRAN_STRT_TS'
			     + '<' + '' + @TargetDate + ''
		end

		/****************************************************************/
		/* Aggregrates: Clear historical tables through PRD_AGGR table.	*/
		/*	EMPL_PURCH_CUR  EMPL_REG_PERF_CUR EMPL_REG_SLS_CUR	*/
		/*	MERCH_SLS_CUR   PLU_SLS_CUR       STR_DEPT_SLS_CUR	*/
		/*	TAX_SLS_CUR     TILL_TND_CUR				*/
		/****************************************************************/

		else if @del_data_type = 'A'
		begin
			-- Delete From <config.table_name> From PRD_AGGR
			-- Where  <config.table_name>.PRD_AGGR_ID = PRD_AGGR.PRD_AGGR_ID
			-- And    <config.tbl_cd> = PRD_AGGR.TBL_CD
			-- And    PRD_AGGR.END_DT has a date less than (today's date - <config.keep_days>)

			select @sql = 'Delete From '
			 + @table_name
			  + ' From PRD_AGGR Where ' 
			   + @table_name
			    + '.PRD_AGGR_ID=PRD_AGGR.PRD_AGGR_ID And '
			     + convert(varchar(10), @tbl_cd)
			      + '=PRD_AGGR.TBL_CD And PRD_AGGR.END_DT'
			       + '<' + '' + @TargetDate + ''
		end

		/****************************************************************/
		/* Receiving: Clear Receipt tables through RECEIPT_PO and PO.	*/
		/*	RECEIP_LOG	RECEIPT_ERROR 	RECEIPT_PKG		*/
		/*	RECEIPT 	RECEIPT_ITEM 	RECEIPT_ITEM_CHG	*/
		/****************************************************************/

		else if @del_data_type = 'R'
		begin
			-- Delete From <config.table_name> From RECEIPT_PO, PO
			-- Where  RECEIPT_PO.PO_SRC = PO.PO_SRC
			-- And    RECEIPT_PO.PO_NBR = PO.PO_NBR
			-- And    <config.table_name>.RCPT_NBR = RECEIPT_PO.RCPT_NBR
			-- And    <config.table_name>.STR_ID = RECEIPT_PO.STR_ID
			-- And    <config.stat> = PO.STAT
			-- And    PO.STAT_DT has a date less than (today's date - <config.keep_days>)

			select @sql = 'Delete From '
			 + @table_name
			  + ' From RECEIPT_PO,PO Where '
			   + 'RECEIPT_PO.PO_SRC=PO.PO_SRC And RECEIPT_PO.PO_NBR=PO.PO_NBR And '
			    + @table_name
			     + '.RCPT_NBR=RECEIPT_PO.RCPT_NBR And '
			      + @table_name
			       + '.STR_ID=RECEIPT_PO.STR_ID And '
 			        + convert(varchar(10), @stat)
			         + '=PO.STAT And PO.STAT_DT'
				  + '<' + '' + @TargetDate + ''
		end

		/****************************************************************/
		/* Purchase Orders: Clear Order tables through PO table.	*/
		/*	PO_SUPPL_ITEM	PO_ITEM_CHG	PO_ITEM			*/
		/*	PO_CHANGE	PO_INSTR_CHG	PO_INSTRUCTION		*/
		/*	PO_OTR							*/
		/****************************************************************/

		else if @del_data_type = 'O'
		begin
			-- Delete From <config.table_name> From PO
			-- Where  <config.table_name>.PO_SRC = PO.PO_SRC
			-- And    <config.table_name>.PO_NBR = PO.PO_NBR
			-- And    <config.stat> = PO.STAT
			-- And    PO.STAT_DT has a date less than (today's date - <config.keep_days>)

			select @sql = 'Delete From '
			 + @table_name
			  + ' From PO Where '
			   + @table_name
			    + '.PO_SRC=PO.PO_SRC And '
			     + @table_name
			      + '.PO_NBR=PO.PO_NBR And '
 			       + convert(varchar(10), @stat)
			        + '=PO.STAT And PO.STAT_DT'
				 + '<' + '' + @TargetDate + ''
		end

		/****************************************************************/
		/* Inventory: Clear Inventory tables through INVENTORY_EVENT.	*/
		/*	INV_ADJUSTMENT	INV_LOCATION	 INV_SELECTION		*/
		/*	INVENTORY_COUNT	INVENTORY_RESULT INVENTORY_SCOPE	*/
		/****************************************************************/

		else if @del_data_type = 'I'
		begin
			-- Delete From <config.table_name> From INVENTORY_EVENT
			-- Where  <config.table_name>.STR_ID = INVENTORY_EVENT.STR_ID
			-- And    <config.table_name>.INV_NBR = INVENTORY_EVENT.INV_NBR
			-- And    <config.stat> = INVENTORY_EVENT.STAT
			-- And    INVENTORY_EVENT.STAT_DT has a date less than (today's date - <config.keep_days>)

			select @sql = 'Delete From '
			 + @table_name
			  + ' From INVENTORY_EVENT Where '
			   + @table_name
			    + '.STR_ID=INVENTORY_EVENT.STR_ID And '
			     + @table_name
			      + '.INV_NBR=INVENTORY_EVENT.INV_NBR And '
 			       + convert(varchar(10), @stat)
			        + '=INVENTORY_EVENT.STAT And INVENTORY_EVENT.STAT_DT'
				 + '<' + '' + @TargetDate + ''
		end

		/****************************************************************/
		/* Transfers: Clear Transfer tables through TRANSFER table.	*/
		/*	TRANSFER_PKG	TRANSFER_ITEM				*/
		/****************************************************************/

		else if @del_data_type = 'X'
		begin
			-- Delete From <config.table_name> From TRANSFER
			-- Where  <config.table_name>.STR_ID = TRANSFER.STR_ID
			-- And    <config.table_name>.TRNF_NBR = TRANSFER.TRNF_NBR
			-- And    <config.stat> = TRANSFER.STAT
			-- And    TRANSFER.STAT_DT has a date less than (today's date - <config.keep_days>)

			select @sql = 'Delete From '
			 + @table_name
			  + ' From TRANSFER Where '
			   + @table_name
			    + '.STR_ID=TRANSFER.STR_ID And '
			     + @table_name
			      + '.TRNF_NBR=TRANSFER.TRNF_NBR And '
			       + convert(varchar(10), @stat)
			        + '=TRANSFER.STAT And TRANSFER.STAT_DT'
				 + '<' + '' + @TargetDate + ''
		end

		/****************************************************************/
		/* Returns To Vendor: Clear RTV tables through RTV table.	*/
		/*	RTV_PKG		RTV_ITEM				*/
		/****************************************************************/

		else if @del_data_type = 'V'
		begin
			-- Delete From <config.table_name> From RTV
			-- Where  <config.table_name>.RTV_SRC = RTV.RTV_SRC
			-- And    <config.table_name>.RTV_NBR = RTV.RTV_NBR
			-- And    <config.stat> = RTV.STAT
			-- And    RTV.STAT_DT has a date less than (today's date - <config.keep_days>)

			select @sql = 'Delete From '
			 + @table_name
			  + ' From RTV Where '
			   + @table_name
			    + '.RTV_SRC = RTV.RTV_SRC And '
			     + @table_name
			      + '.RTV_NBR = RTV.RTV_NBR And '
			       + convert(varchar(10), @stat)
			        + '=RTV.STAT And RTV.STAT_DT'
				 + '<' + '' + @TargetDate + ''
		end

		/****************************************************************/
		/* Intercept the unknown data type.				*/
		/****************************************************************/

		else
		begin
			select @MsgText = @AppName + ': Unknown ''%1'' data type'
			exec sp_LogMsg 50002, @MsgText, Error, @del_data_type
			select @rc = -2
			break
		end

		/****************************************************************/
		/* If any, append the further rule to the SQL statement.	*/
		/****************************************************************/

		if datalength(@sql_rule) > 1
		begin
			select @sql2 = ' And ' + @sql_rule
		end

		/****************************************************************/
		/* Execute		 					*/
		/****************************************************************/

		if @DebugFlag = 0
		begin
			exec (@sql + @sql2)
			select @rowcount = @@ROWCOUNT, @error = @@ERROR

			while @rowcount <> 0 and @error = 0
			begin
				select @rows = @rows + @rowcount
				exec (@sql + @sql2)
				select @rowcount = @@ROWCOUNT, @error = @@ERROR
			end
		end
		else select @sql + @sql2

		/****************************************************************/
		/* Check error.		 					*/
		/****************************************************************/

		if @error <> 0
		begin
			select @MsgText = @AppName + ': Error ''' + convert(varchar(16), @error)
					   + ''' deleting ''%1'' table. DelDataType=%2'
			exec sp_LogMsg 50003, @MsgText, Error, @table_name, @del_data_type
			select @rc = -3
			break
		end
		else
		begin
			select @MsgText = @table_name + ' Cleaned successfully'
--CarloC		exec sp_LogMsg 50012, @MsgText, Informational
		end

		FETCH NEXT FROM del_cursor INTO @table_name, @del_data_type, @day_of_week, @column_name, @keep_days, @tbl_cd, @stat, @sql_rule, @sp_flag
	end

	CLOSE del_cursor
	DEALLOCATE del_cursor

	/****************************************************************/
	/* Exit.							*/
	/****************************************************************/

	if @rc <> 0
	begin
		select @MsgText = @AppName + ' terminated with error(s)'
		exec sp_LogMsg 50011, @MsgText, Error
	end
	else
	begin
		select @MsgText = @AppName + ' terminated successfully'
		exec sp_LogMsg 50012, @MsgText, Informational
	end

	return @rc
END

GO
SET QUOTED_IDENTIFIER OFF 
GO
SET ANSI_NULLS ON 
GO


```



清理条件的数据结构

``` sql
CREATE TABLE [DEL_DATA_CONFIG] (
	[GROUP_ID] [smallint] NOT NULL ,
	[SEQ_ID] [smallint] NOT NULL ,
	[TABLE_NAME] [varchar] (30) COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL ,
	[DEL_DATA_TYPE] [char] (1) COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL ,
	[DAY_OF_WEEK] [varchar] (7) COLLATE SQL_Latin1_General_CP1_CI_AS NOT NULL ,
	[COLUMN_NAME] [varchar] (30) COLLATE SQL_Latin1_General_CP1_CI_AS NULL ,
	[KEEP_DAYS] [smallint] NULL ,
	[TBL_CD] [smallint] NULL ,
	[STAT] [smallint] NULL ,
	[SQL_RULE] [varchar] (255) COLLATE SQL_Latin1_General_CP1_CI_AS NULL ,
	[SP_FG] [smallint] NULL CONSTRAINT [DF__DEL_DATA___SP_FG__60A75C0F] DEFAULT (0),
	CONSTRAINT [PK_DelDataConfig] PRIMARY KEY  CLUSTERED 
	(
		[GROUP_ID],
		[SEQ_ID]
	)  ON [PRIMARY] 
) ON [PRIMARY]
GO


```

