# Service Fabric with a Statless ASP.NET Core 2.2 application that connects to a local db

This template is a quick-and-dirty way to have Service Fabric correctly read from a local SQL db.

Create a new Service Fabric project.
-------------------------------------

* Run Visual Studio in Administrator mode
* Select Service Fabric and choose a Statless ASP.NET Core Web Application with Identity.
* The template in this repo is a .Net Core 2.2 and I am using Visual studio 2019

```c#
app.UseMvc(routes =>
{
     routes.MapRoute(
          name: "default",
          template: "{controller=Home}/{action=Index}/{id?}");
});
```
