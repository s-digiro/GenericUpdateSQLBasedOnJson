# GenericUpdateSQLBasedOnJson
Updates SQL database based on given json string

USE [MyDatabase]
GO
/****** Object:  StoredProcedure [dbo].[ProcessDatabaseActions]    Script Date: 04/01/21 11:34:27 ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

ALTER PROCEDURE [dbo].[ProcessDatabaseActions]
	@databaseactions varchar(max) = ''
AS
BEGIN
	SET NOCOUNT ON;
	SELECT * INTO #temp from [dbo].[parseJSON](@databaseactions);

	DECLARE @root AS INT = (SELECT t.Object_ID FROM #temp t WHERE t.Parent_ID = NULL);

	DECLARE @updates AS INT = (SELECT t.Object_ID FROM #temp t WHERE t.Parent_ID = @root AND t.Name = 'updates');
	DECLARE @inserts AS INT = (SELECT t.Object_ID FROM #temp t WHERE t.Parent_ID = @root AND t.Name = 'inserts');
	DECLARE @deletes AS INT = (SELECT t.Object_ID FROM #temp t WHERE t.Parent_ID = @root AND t.Name = 'deletes');

	-- Iterate over updates, inserts, and then deletes
	DECLARE @current INT = @updates;
	DECLARE @finished BIT = 0;
	WHILE @finished = 0
	BEGIN
		DECLARE OP_LIST CURSOR FOR SELECT t.SequenceNo, t.Object_ID FROM #temp t WHERE t.Parent_ID = @current ORDER BY t.SequenceNo;

		OPEN OP_LIST;
		DECLARE @seqNo as INT;
		DECLARE @op as INT;
		FETCH NEXT FROM OP_LIST INTO @seqNo, @op;
		WHILE @@FETCH_STATUS = 0
		BEGIN
			DECLARE @targetTable AS VARCHAR(100) = (SELECT t.StringValue FROM #temp t WHERE t.Parent_ID = @op AND t.Name = 'table');
			
			DECLARE ACTIONS CURSOR FOR SELECT t.SequenceNo, t.Object_ID FROM #temp t WHERE t.Parent_ID = @op ORDER BY t.SequenceNo;

			OPEN ACTIONS
			DECLARE @actionSeqNo as INT;
			DECLARE @action as INT;
			FETCH NEXT FROM ACTIONS INTO @actionSeqNo, @action;
			WHILE @@FETCH_STATUS = 0
			BEGIN
				DECLARE @keys AS INT = (SELECT t.Object_ID FROM #temp t WHERE t.Parent_ID = @updates AND t.Name = 'keys');
				DECLARE KEYS_LIST CURSOR FOR SELECT t.Name, t.StringValue FROM #temp t WHERE t.Parent_ID = @keys;

				DECLARE @items AS INT = (SELECT t.Object_ID FROM #temp t WHERE t.Parent_ID = @updates AND t.Name = 'item');
				DECLARE ITEMS_LIST CURSOR FOR SELECT t.Object_ID, t.Name, t.StringValue, t.valueType FROM #temp t WHERE t.Parent_ID = @items;

				DECLARE @query AS VARCHAR(max);

				-- Update
				IF(@current = @updates)
				BEGIN
					SET @query  = 'UPDATE ' + @targetTable + ' SET ';

					OPEN ITEM_LIST;
					DECLARE @item AS INT;
					DECLARE @name AS VARCHAR(max);
					DECLARE @stringValue AS VARCHAR(max);
					FETCH NEXT FROM ITEM_LIST INTO @item, @name, @stringValue
					WHILE @@FETCH_STATUS = 0
					BEGIN
						SET @query = @query + @name + ' = ' + @stringValue;
						FETCH NEXT FROM ITEMS INTO @item, @name, @stringValue;
						IF(@@FETCH_STATUS = 0)
							SET @query = @query + ', ';
					END
					SET @query = @query + ' WHERE ';

					OPEN KEYS;
					DECLARE @key AS INT;
					FETCH NEXT FROM KEYS INTO @key, @name, @stringValue
					WHILE @@FETCH_STATUS = 0
					BEGIN
						SET @query = @query + @name + ' = ' + @stringValue;
						FETCH NEXT FROM ITEMS INTO @key, @name, @stringValue
						IF(@@FETCH_STATUS = 0)
							SET @query = @query + ' AND ';
					END
					SET @query = @query + ';';
				END

				-- Insert
				ELSE IF(@current = @inserts)
				BEGIN
					SET @query = 'INSERT INTO ' + @targetTable + ' (';
					DECLARE @values AS VARCHAR(max) = 'VALUES (';

					OPEN ITEM_LIST;
					FETCH NEXT FROM ITEM_LIST INTO @item, @name, @stringValue;
					WHILE @@FETCH_STATUS = 0
					BEGIN
						SET @query = @query + @name;
						SET @values = @values + @stringValue;
						FETCH NEXT FROM ITEM_LIST INTO @item, @name, @stringValue;
						IF(@@FETCH_STATUS = 0) SET @query = @query + ', ';
						IF(@@FETCH_STATUS = 0) SET @values = @values + ', ';
					END
					SET @query = @query + ') ' + @values + ');';
				END
				
				-- Delete
				ELSE IF(@current = @deletes)
				BEGIN
					SET @query = 'DELETE FROM ' + @targetTable + ' WHERE ';
	
					OPEN KEYS;
					FETCH NEXT FROM KEYS INTO @key, @name, @stringValue;
					WHILE @@FETCH_STATUS = 0
					BEGIN
						SET @query = @query + @name + '=' + @stringValue;
						FETCH NEXT FROM ITEMS INTO @key, @name, @stringValue;
						IF(@@FETCH_STATUS = 0) SET @query = @query + ' AND ';
					END
					SET @query = @query + ';';
				END

				EXEC(@query);
			END
		END

		IF(@current = @updates)
			SET @current = @inserts
		ELSE IF(@current = @inserts)
			SET @current = @deletes
		ELSE IF(@current = @deletes)
			SET @finished = 1;
	END
END

DECLARE @dbActions AS VARCHAR(MAX) = '	{
	"deletes": [],
	"updates": [{
		"table": "Jobs",
		"actions": [{
			"keys": {
				"JobId": 31028
			},
			"item": {
				"JobId": 31028
				"Job": "A Simple Job",
				"Rounded": true
			}
		}, {
			"keys": {
				"JobId": 31038
			},
			"item": {
				"JobId": 31038,
				"Job": "A Tough Job",
				"Rounded": false
			}
		}]
	}],
	"inserts": [{
		"table": "SomeTable",
		"actions": [{
			"keys": {
				"FirstKey": 99991,
				"SecondKey": "333"
			},
			"item": {
				"FirstKey": 99991,
				"SecondKey": "333",
				"Status": "C",
				"Affects_Schedule": true,
				"Certs Required": false,
				"Stocked_UofM": "each"
			}
		}]
	}]
}';

EXEC ProcessDatabaseActions @databaseactions=@dbActions;
