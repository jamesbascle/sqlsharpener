# SqlSharpener
Rather than generating code from the database or using a heavy abstraction layer that might miss differences between the database and data access layer until run-time, this project aims to provide a very fast and simple data access layer that is generated at design-time using SQL _files_ as the source-of-truth (such as those found in an SSDT project).

SqlSharpener accomplishes this by parsing the SQL files to create a meta-object hierarchy with which you can generate C# code such as stored procedure wrappers or Entity Framework Code-First entities. You can do this manually or by invoking one of the included pre-compiled T4 templates.

# Example

Here is one way you can use SqlSharpener...

**Given a stored procedure that looks like this:**

    CREATE PROCEDURE [dbo].[usp_TaskGet]
    	@TaskId int
    AS
  	-- Specifying "TOP 1" makes the generated return value a single instance instead of an array.
  	SELECT TOP 1
  		t.Name, t.[Description], t.Created, t.CreatedBy, t.Updated, t.UpdatedBy,
        ts.Name as [Status]
  	FROM Tasks t
  	JOIN TaskStatus ts ON t.TaskStatusId = ts.Id
  	WHERE t.Id = @TaskId

**The included pre-compiled Stored Procedure template will generate C# code like this:**

    // ------------------------------------------------------------------------------
    // <auto-generated>
    //     This code was generated by SqlSharpener.
    //  
    //     Changes to this file may cause incorrect behavior and will be lost if
    //     the code is regenerated.
    // </auto-generated>
    // ------------------------------------------------------------------------------
    namespace SimpleExample.DataLayer
    {
        using System;
        using System.IO;
        using System.Data;
        using System.Data.SqlClient;
        using System.Configuration;
        using System.Collections.Generic;

        /// <summary>
        /// Interface of the wrapper class for calling stored procedures. 
        /// </summary>
        public partial interface IStoredProcedures
        {
            Result<TaskGetDto> TaskGet( Int32? TaskId );
        }

        /// <summary>
        /// Wrapper class for calling stored procedures. 
        /// </summary>
        public partial class StoredProcedures : IStoredProcedures
        {
            private string connectionString;

            public StoredProcedures(string connectionString)
            {
                this.connectionString = connectionString;
            }

            /// <summary>
            /// Calls the "usp_TaskGet" stored procedure
            /// </summary>
            /// <returns>A DTO filled with the results of the SELECT statement.</returns>
            public Result<TaskGetDto> TaskGet( Int32? TaskId )
            {
                OnTaskGetBegin();
                Result<TaskGetDto> result = new Result<TaskGetDto>();
                using(var conn = new SqlConnection(connectionString))
                {
                    conn.Open();
                    using (var cmd = conn.CreateCommand())
                    {
                        cmd.CommandType = CommandType.StoredProcedure;
                        cmd.CommandText = "usp_TaskGet";
                        cmd.Parameters.Add("TaskId", SqlDbType.Int).Value = (object)TaskId ?? DBNull.Value;

                        using(var reader = cmd.ExecuteReader(CommandBehavior.SequentialAccess))
                        {
                            result.RecordsAffected = reader.RecordsAffected;
                            while (reader.Read())
                            {
                                var item = new TaskGetDto();
                                item.Name = reader.GetString(0);
                                item.Description = reader.GetString(1);
                                item.Status = reader.GetString(2);
                                item.Created = reader.GetDateTime(3);
                                item.CreatedBy = reader.GetString(4);
                                item.Updated = reader.GetDateTime(5);
                                item.UpdatedBy = reader.GetString(6);
                                result.Data = item;
                            }
                            reader.Close();
                        }
                    }
                    conn.Close();
                }
                OnTaskGetEnd(result);
                return result;
            }
            partial void OnTaskGetBegin();
            partial void OnTaskGetEnd(Result<TaskGetDto> result);
        }

        /// <summary>
        /// The return value of the stored procedure functions.
        /// </summary>
        public partial class Result<T>
        {
            public T Data { get; set; }
            public int RecordsAffected { get; set; }
        }

        /// <summary>
        /// DTO for the output of the "usp_TaskGet" stored procedure.
        /// </summary>
        public partial class TaskGetDto 
        {
            public String Name { get; set; }
            public String Description { get; set; }
            public String Status { get; set; }
            public DateTime Created { get; set; }
            public String CreatedBy { get; set; }
            public DateTime Updated { get; set; }
            public String UpdatedBy { get; set; }
        }
    }


# Performance

According to the performance tests from [Dapper](https://github.com/StackExchange/dapper-dot-net#performance-of-select-mapping-over-500-iterations---poco-serialization), the best performance came from hand-coded functions using SqlDataReader. SqlSharpener gives you that performance without needing to hand-code your functions.

# Installation

[Using NuGet](https://www.nuget.org/packages/SqlSharpener/), run the following command to install SqlSharpener:

    PM> Install-Package SqlSharpener
    
This will add SqlSharpener as a solution-level package. That means that the dll's do not get added to any of your projects (nor should they). 

# Quick Start

See the [Quick Start Guide](https://github.com/aeslinger0/sqlsharpener/wiki/Quick-Start-Guide) for more examples.

The fastest way to get up and running is to call one of SqlSharpener's included pre-compiled templates from your template. Add a new T4 template (\*.tt) file to your data project and set its content as follows: *(Ensure you have the correct version number in the dll path)*

    <#@ template debug="false" hostspecific="true" language="C#" #>
    <#@ assembly name="$(SolutionDir)\packages\SqlSharpener.1.0.2\tools\SqlSharpener.dll" #>
    <#@ output extension=".cs" #>
    <#@ import namespace="Microsoft.VisualStudio.TextTemplating" #>
    <#@ import namespace="System.Collections.Generic" #>
    <#@ import namespace="SqlSharpener" #>
    <#
    	// Specify paths to your *.sql files. Remember to include your tables as well! We need them to get the data types.
    	var sqlPaths = new List<string>();
    	sqlPaths.Add(Host.ResolvePath(@"..\SimpleExample.Database\dbo\Tables"));
    	sqlPaths.Add(Host.ResolvePath(@"..\SimpleExample.Database\dbo\Stored Procedures"));
    
    	// Set parameters for the template.
    	var session = new TextTemplatingSession();
    	session["outputNamespace"] = "SimpleExample.DataLayer";
    	session["procedurePrefix"] = "usp_";
    	session["sqlPaths"] = sqlPaths;
    
    	// Generate the code.
    	var t = new SqlSharpener.StoredProceduresTemplate();
        t.Session = session;
    	t.Initialize();
    	this.Write(t.TransformText());
    #>

The generated .cs file will contain a class with functions for all your stored procedures, DTO objects for procedures that return records, and an interface you can used if you use dependency-injection. Whenever your database project changes, simply right-click on the .tt file and click "Run Custom Tool" to regenerate the code.

# Usage

Once the code is generated, your business layer can call it like any other function. Here is one example:

        public TaskGetDto Get(int id)
        {
            return storedProcedures.TaskGet(id);
        }
        
# Dependency Injection

If you use a dependency-injection framework such as Ninject, you can use the interface generated. For example:

    public class  DataModule : NinjectModule
    {
        public override void Load()
        {
            Bind<IStoredProcedures>().To<StoredProcedures>();
        }
    }
    
# Documentation

Check out the [wiki](https://github.com/aeslinger0/sqlsharpener/wiki) for more info.
    
# License

SqlSharpener uses The MIT License (MIT), but also has dependencies on DacFx and ScriptDom. I have included their license info in the root directory.
