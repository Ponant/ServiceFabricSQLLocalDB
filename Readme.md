# Service Fabric with a Statless ASP.NET Core 2.2 application that connects to a local db

This template is a quick-and-dirty way to have Service Fabric correctly read from a local SQL db.
It is related to the issue https://github.com/aspnet/EntityFrameworkCore/issues/13969

Create a new Service Fabric project.
-------------------------------------

* Run Visual Studio in Administrator mode
* Select Service Fabric and choose a Statless ASP.NET Core Web Application with Identity.
* The template in this repo is a .Net Core 2.2 under Visual Studio 2019 Preview

Run the project and first bugs
-------------------------------------

If you run this project you will get this error
`ArgumentNullException: Value cannot be null.
Parameter name: connectionString
Microsoft.EntityFrameworkCore.Utilities.Check.NotEmpty(string value, string parameterName)`

If you set the breakpoint in `Startup.cs` you will see that
`Configuration.GetConnectionString("DefaultConnection")` is null.
That is due to a misconfigruation of the Service Fabric template when one of the microservice is a stateless ASP.NET Core app
with an Identity database. As a temporary workaround, we circumvent reading from
appsettings by a direct copy and paste of the `DefaultConnection` string in `Startup.cs`

```c#
var DefaultConnection = "Server=(localdb)\\mssqllocaldb;Database=WebIdentityDb;Trusted_Connection=True;MultipleActiveResultSets=true";

services.AddDbContext<ApplicationDbContext>(options =>
              options.UseSqlServer(DefaultConnection));
```

Now the app runs on the home page, but if you navigate to the
register page and register a new user you will get another throw:

`Win32Exception: Unknown error (0x89c50118)
Unknown location
SqlException: A network-related or instance-specific error occurred while establishing a connection to SQL Server. The server was not found or was not accessible. Verify that the instance name is correct and that SQL Server is configured to allow remote connections. (provider: SQL Network Interfaces, error: 50 - Local Database Runtime error occurred. Cannot create an automatic instance. See the Windows Application event log for error details.
)`

It seems the issues comes from how Service Fabric implements permissions. The best answer was found here
https://stackoverflow.com/questions/36401991/using-localdb-with-service-fabric

From a command prompt, run 
`sqllocaldb share MSSqlLocalDB SharedDB`

Update the configuration string to

`var DefaultConnection = @"Data Source=(localdb)\.\SharedDB;Initial Catalog=WebIdentityDb;Integrated Security=true;";`

and configure the `SQL Server Management Studio` following the link above. Also see
https://www.youtube.com/watch?v=1wADjVteNa8 which was usefull in setting up `db_owner`.
This part was the most difficult one for me as I had little experience in permissions of SQL dbs.

If you now run the application and register you will get the usual prompt to apply migrations and update the database. But an error will occur saying the migrations could not be applied.
You won't ba able to do that from a command line. The reason is probably because that EF tools are not configured in the Service Fabric template.
The most brutal way of doing this is to set in `Program.cs` of the statless ASP:ET Core project:

```c#
private static IWebHost BuildWebHost(string[] args)
{
    return WebHost.CreateDefaultBuilder(args).UseStartup<Startup>().Build();
}
```

Then set the stateless ASP.NET Core Identity project as a startup project. Open the console package manager and set the ASP.NET Cora app
as the default project and apply the migration and update the database as usual. 

`add-migration initial`

`update-database`

Reset the Service Fabric project to be the startup project and run the app. It should now work.

Once that done you can move the DefaultConnection to be read from the settings files of the Service Fabric app and ultimately Azure Key Vault. You can also remove the private method returning `IWebHost`
to configure explicitly through extension methods in `WebIdentity.cs` to invoke EF tooling and possibly reading from `appsettings.json` if that is needed.

Let me know if you have some clever improvments, I am aware that is only a temporary solution.
