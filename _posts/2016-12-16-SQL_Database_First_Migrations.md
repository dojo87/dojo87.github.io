---
layout: post
title: SQL Database First Migrations
published: true
disqus: true
---


# What's the problem
Probably database-first is still a frequent way of approaching databases. I had encountered some projects in my life that had some manual-driven, handy work done to make an update. One project (that was still before it's production stage) had something like a "change-backup-restore" cycle to make the changes. It was like this:
- somebody wanted to make a change on a database - he shouted to the team, that he will be doing so (yep, that is a "lock" on the database mechanism)
- the person firstly restored a last backup of the database (so the changes made were clean), done his changes...
- ...and made a backup which was then commited to SVN
- the person said, he has released the "lock".
- everybody downloaded the backup and restored on their version.

It would be probably easier to have at least a shared Dev database, right?. The same was applied to the UAT environment, because there was no production at that time. I've came in to the project after a few months of this madness and... the client wanted to go Live! 

Before we have gone to a state of automatic updates, there was some time of adding scripts to source control, which the "deployer" will execute one by one - manually. With that approach we were safe, but loaded with a lot of manual work and a little pinch of stress on deployments. It had to stop.

# What's the idea
What ideas developers have for solving this? 
- the one I've described - manual scripts ( bad idea)
- SQL compare tools (like Redgate SQL Compare - http://www.red-gate.com/products/sql-development/sql-compare/ ) - not bad, but still some manual work. What about some test stuff developers insert in a database? What if you need to make an explicit insert to the production?
- How about ... database-first migrations?

Code-First approach has this cherry at the top - Migrations. Run Add-Migration after defining your class, and voila - you get a fully-automatic migration class with Up and Down methods so you can move anywhere in the migration flow. The concept of a migration is that you divide your changes into iterations (even the initial DB version) and execute them one by one as your project grows and changes.

Why not apply this approach to Database-First? The migration can be a SQL script, which will also have a number and will be executed in order. Making an Up/Down method isn't trivial, but having a Up script that is executed in an automatic manner is already a big jump forward. 

# What's the solution
This is only a portion of code. The full project ready to use under MIT can taken from
NuGet: https://www.nuget.org/packages/Jamrozik.SqlForward
and source code at GitHub: https://github.com/dojo87/Jamrozik.SqlForward

```csharp
public class DatabaseMigration
{
        /// <summary>
        /// Initializes a new instance of the DatabaseMigration class specifying a factory function for the database connection.
        /// </summary>
        /// <param name="connectionFactory">The connection factory is a function that returns a constructed object 
        /// of interface IDbConnection. It ensures, that the connection is used only by the migrator
        /// and no other function, because the connection returned by the connectionFactory is
        /// used and disposed after the migration process. The connection isn't reusable.</param>
        public DatabaseMigrationMinimal(Func<IDbConnection> connectionFactory)
        {
            this.ConnectionFactory = connectionFactory;
            if (this.ConnectionFactory == null){
                throw new ArgumentNullException("No connection factory provided");
            }
        }

        #region Properties
        
        /// <summary>
        /// Gets or privetly sets the ConnectionFactory creating the IDbConnection instance.
        /// </summary>
        protected Func<IDbConnection> ConnectionFactory
        {
            get;
            private set;
        }

        /// <summary>
        /// Holds the Connection created by the Connection factory.
        /// </summary>
        protected IDbConnection Connection
        {
            get;
            private set;
        }

        /// <summary>
        /// Gets or sets the folder containing migration scripts.
        /// Default: SqlForward.DatabaseScripts from AppConfig AppSettings.
        /// </summary>
        public string DatabaseScripts
        {
            get;
            set;
        } = ConfigurationManager.AppSettings["SqlForward.DatabaseScripts"];

        /// <summary>
        /// Gets or sets the initialization script file name (is assumed to be in the DatabaseScripts folder).
        /// Default: SqlForward.Initialization from AppConfig AppSettings.
        /// </summary>
        public string InitializationScript
        {
            get;
            set;
        } = ConfigurationManager.AppSettings["SqlForward.Initialization"];
}
```

At this point it is self explanatory. I've choosen to make the IDbConnection (main interface for .NET connections) provider as a factory function rather than explicitly string, to make it resolvable at runtime. 

We have to point also to some script source - a folder containing the migration SQL files (DatbaseScripts) and additionally the name of the script for initializing a table, which will have the same role like the MigrationHistory table in Code-First migrations.

Lets assume an initialization file "Initialization.sql":

```sql
IF (NOT EXISTS (SELECT * FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA = 'dbo' AND  TABLE_NAME = 'ScriptLog'))
CREATE TABLE [dbo].[ScriptLog](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[ScriptName] [nvarchar](200) NULL,
	[ScriptDate] [datetime] NULL
 CONSTRAINT [PK_ScriptLog] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX  = OFF, STATISTICS_NORECOMPUTE  = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS  = ON, ALLOW_PAGE_LOCKS  = ON) ON [PRIMARY]
) ON [PRIMARY]

IF NOT EXISTS (SELECT * FROM dbo.ScriptLog WHERE ScriptName = 'Initialization.sql')
BEGIN
INSERT INTO ScriptLog (ScriptName, ScriptDate, Status, DomainUser, Application) VALUES ('Initialization.sql',GETDATE());
END
```

Keeping the SQL files that were already executed on a DB instance IN that very same instance will give us great versioning control throughout environments (dev, uat, prod, etc.). We explicitly know which were executed on a particular instance by having them enumerated in that instance, in a special table.

We will execute every migration script in a transaction, to keep it consistent. If a certain migration will fail, it will be rollbacked and the whole process will stop to review the errors. We cannot execute other migrations, because that would make the DB inconsistent.

```csharp
protected virtual void ExecuteScriptInTransaction(string script)
{
    using (IDbTransaction transaction = this.Connection.BeginTransaction())
    {
        string migrationName = Path.GetFileNameWithoutExtension(script);
        Stopwatch watch = new Stopwatch();
        try
        {
            watch.Start();
            string scriptCommand = File.ReadAllText(script);
            IDbCommand initializationCommand = this.Connection.CreateCommand();
            initializationCommand.CommandText = scriptCommand;
            initializationCommand.Transaction = transaction;
            initializationCommand.ExecuteNonQuery();
            transaction.Commit();

        }
        catch (Exception ex)
        {
            // More sophisticated error tracking here :)
            Trace.WriteLine($"Error on migration {migrationName} (time: {watch.Elapsed}) {ex}"); 
            transaction.Rollback();
            throw; 
        }
    }
}
```

Lets define the initialization of our ScriptLog table:

```csharp
private void InitializeScriptLogTable()
{
    string scriptLogCreation = Path.Combine(this.DatabaseScripts, InitializationScript);
    ExecuteScriptInTransaction(scriptLogCreation);
}
```

Now to execute pending migrations I need to get the migration files contained in the folder and the migrations that are already applied to the database. After comparing those lists we will have pending migrations.

```csharp
private List<string> GetExecutedScripts()
{
    IDbCommand executedScriptsCommand = this.Connection.CreateCommand();
    executedScriptsCommand.CommandText = @"SELECT [ScriptName],[ScriptDate] FROM ScriptLog";

    List<string> executedScriptNames = new List<string>();
    IDataReader reader = executedScriptsCommand.ExecuteReader();
    while (reader.Read())
    {
        executedScriptNames.Add(reader.GetString(0));
    }
    reader.Close();
    return executedScriptNames;
}

private List<string> GetScriptsToExecute()
{
    var files = new List<string>(Directory.GetFiles(this.DatabaseScripts, "*.sql"));
    files.Sort();
    return files;
}
```

After executing a migration file we will have to record that fact in our ScriptLog:

```csharp
private void RecordMigrationExecuted(string fileName)
{
    IDbCommand insertScriptLog = this.Connection.CreateCommand();
    insertScriptLog.CommandText = "INSERT INTO ScriptLog(ScriptName, ScriptDate) VALUES(@name, GETDATE());";
    AddParameterToCommand(insertScriptLog, DbType.String, "name", fileName);
    insertScriptLog.ExecuteNonQuery();
}

//helper method:
private IDataParameter AddParameterToCommand(IDbCommand command, DbType type, string name, object value)
{
    IDataParameter parameter = command.CreateParameter();
    parameter.ParameterName = name;
    parameter.DbType = type;
    parameter.Value = value;
    command.Parameters.Add(parameter);
    return parameter;
}
```

Lets combine that. We will iterate through migrations and record every executed migration:

```csharp
// Main Entry point to start executing pending migrations.
public void Synchronize()
{
    using (this.Connection = this.ConnectionFactory())
    {
        if (this.Connection.State == ConnectionState.Closed)
        {
            this.Connection.Open();
        }

        InitializeScriptLogTable();

        IterateMigrations(GetExecutedScripts(), GetScriptsToExecute());
    }
}

private void IterateMigrations(List<string> executedScriptNames, List<string> files)
{
    foreach (var file in files)
    {
        string fileName = Path.GetFileName(file);
        if (!executedScriptNames.Contains(fileName))
        {
            ExecuteScriptInTransaction(file);
            RecordMigrationExecuted(fileName);
        }
    }
}
```

This is a simple version, but lets consider some more improvements (which I hope to add to the NuGet package!)
- extending the ScriptLog table (giving it a generic structure that the Developer can define), so that he can log more information like executing user, application name).
- adding dynamic parameters to the execution, so that we can make use of any app settings or other dynamic variables.
- extending the migration naming convention to contain an environment variable e.g. 20161216_MyMigration.Production.sql, so we can differentiate between Test, Production, Dev (probably used for some specific configuration inserts that would be different on different environments).
- extend error and event loging
- extend Synchronize to take a specific migration to go UP to.
- extend the migration file naming convention to allow "Down" migrations? 

Be sure to track any updates on GitHub or NuGet:
- NuGet: https://www.nuget.org/packages/Jamrozik.SqlForward
- GitHub: https://github.com/dojo87/Jamrozik.SqlForward
