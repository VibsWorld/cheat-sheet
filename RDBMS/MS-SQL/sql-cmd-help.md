## SQL Commands (running via `sqlcmd`)

Check if a database exists. 

Option 1: Quick one-liner
`sqlcmd -S localhost -E -Q "IF DB_ID('YourDatabaseName') IS NOT NULL PRINT 'Exists' ELSE PRINT 'Does not exist'"`

Option 2: Query the system catalog
`sqlcmd -S localhost -E -Q "SELECT name FROM sys.databases WHERE name = 'YourDatabaseName'"`

If it returns a row, the database exists. If empty, it doesn't.
Option 3: List ALL databases (to browse)
`sqlcmd -S localhost -E -Q "SELECT name FROM sys.databases ORDER BY name"`
```
