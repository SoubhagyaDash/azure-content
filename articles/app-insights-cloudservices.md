<properties
   pageTitle="Application Insights for Azure Cloud Services"
   description="Monitor your web and worker roles effectively with Application Insights"
   services="application-insights"
   documentationCenter="na"
   authors="soubhagyadash"
   manager="victormu"
   editor="alancameronwills"/>

<tags
   ms.service="application-insights"
   ms.devlang="na"
   ms.topic="article"
   ms.workload="cloud services"
   ms.date="05/12/2015"
   ms.author="sdash"/>

# Application Insights for Azure Cloud Services

**DRAFT -- TBD items**
* Add images
* Fix links to sample code


*Application Insights is in preview*

Application Insights lets you monitor your applications availability, performance, failures and usage. The 360° insights are complete with client side and dependency telemetry. 
As with any application running on .NET framework version 4.0+, Azure Cloud Services are also supported. While some of the tooling that make onboarding a breeze for ASP.NET web apps or Azure web apps etc., is not available at this moment - you can get started with 
a few steps shown below and get all the same goodness. 

Application Insights come in the shape of the following telemetry primitives. The following sections will go over how each can be collected from web and worker roles. 
For illustration purposes, we have altered this [sample Azure cloud service](samplelink) to onboard it. This code is available [here](git link), for you to follow along with the steps below.
For starters, lets cover the basics:
## Basics
<!-- TBD: Add images to help illustrate these -->
* Create a resource on the portal.
  * You will need an Azure subscription 
  * You need to create the AI resources which will essentially be containers for the telemetry your application generates
  * You can manage roles/environments to AI resource mappings however you like: You could have a separate resource for each role & environment (recommended) or manage an AI resource per logical group of resources
  * copy the instrumentation key from the portal <!-- TBD: Add images to help illustrate these -->
* Environment support: To collect AI telemetry from multiple environments (DEV/INT/Pre-Prod/PROD etc): 
  * Set the Instrumentation key in the respective CSCFG files
  * Configure it at application start up time, in the global.asax.cs file for web roles as shown [here](Global.asax.cs#L27)
  * The JavaScript can also read from the same as shown [here](Views/Shared/_Layout.cshtml#L9). 
    * Note that this has a slight performance overhead, but is helpful if you are looking to report both client and server side telemetry to the same instrumentation key

* Nuget packages
  * TBD: Add link to any appropriate documentation on the different nuget packages
  * We encourage you to add the [Application Insights for Web] (http://www.nuget.org/packages/Microsoft.ApplicationInsights.Web) nuget for *both* web and worker roles - as that adds modules that add server context like the Role information etc.

Moving on to the different telemetry types:

##Request Telemetry

####Web Roles
* Collected out of the box with the [Application Insights for web nuget](http://www.nuget.org/packages/Microsoft.ApplicationInsights.Web)
* A request is considered failed if the response has a statusCode >= 400. If that does not work for you (401s are not failures for instance), you can override the default behavior:
  * Provide a custom implementation of the ITelemetryInitializer interface - as shown [here](Telemetry/MyTelemetryInitializer.cs)
  * Add this to the list of TelemetryInitializers in ApplicationInsights.config as shown [here](ApplicationInsights.config#L66)
* Requests are groupable by the "Request Name" attribute, which allows us to provide meaningful aggregations on the number of calls, response times, failures etc. The default naming scheme is the following:
  * ASP.NET MVC: Request name is set to “VERB controller/action”.
  * ASP.NET MVC Web API: Per above both requests “/api/movies/” and “/api/movies/5” will be result in “GET movies”. To support Web API better, request name includes the list of all names of routing parameters if “action” parameter wasn’t found. In this case, the requests will be reported as “GET movies” and “GET movies[id]”.
  * If routing table is empty or doesn’t have “controller” - HttpRequest.Path will be used as a request name. This property doesn’t include domain name and query string.
  * NOTE: Request names are case-sensitive. If the default rules do not work for your application (each request gets a unique name for instance) - you can fix that by providing a custom WebOperationNameTelemetryInitializer implementation to override default behavior.

####Worker Roles
* A request is in essence a unit of *named* server side work that can be *timed* and independently *succeed/fail*
  * As such, Requests can be applied for worker roles, and provides a handy way to capture performance and success telemetry of the different operations a worker role may be doing.
  * For instance, one of the operations in this worker role is to check an Azure table for items to process periodically. That operation is reported as a [CheckItemsTable](WorkerRoleA.cs#L96) request when that completes successfully.
  * When the operation fails with an exception (or any other condition for that matter), you can report a failed request as shown [here](WorkerRoleA.cs#L108)

##Dependency Telemetry

* NOTE: ASYNC HTTP/SQL calls are automatically collected if you are running on .NET Framework versions 4.5.1 or higher
* To collect both SYNC and ASYNC dependency calls with richer detail like SQL statements - set up the [Application Insights Agent](http://azure.microsoft.com/en-us/documentation/articles/app-insights-monitor-performance-live-website-now/) AKA "Status Monitor" with the following steps:
* To use the AI Agent with your web/worker roles:
  * Add the [AppInsightsAgent](AppInsightsAgent) folder and the 2 files in it to your web/worker role projects. Be sure to set them up to be copied always into the output directory
  * Add the start up task to the CSDEF file as shown [here](../AzureEmailService/ServiceDefinition.csdef#L24)
  * For worker roles, add 3 environment variables as shown [here](../AzureEmailService/ServiceDefinition.csdef#L34)
  * For web roles, IIS will be restarted after the agent is successfully installed
  * The agent installation script goes no-op when in emulator mode, so your F5 sessions will not be adversely affected by adding these

##Exception Telemetry

####Web Roles
* This web role has MVC5 and Web API 2 controllers. The unhandled exceptions from the 2 are captured with the following:
  * [AiHandleErrorAttribute](Telemetry/AiHandleErrorAttribute.cs) set up [here](App_Start/FilterConfig.cs#L12) for MVC5 controllers
  * [AiWebApiExceptionLogger](Telemetry/AiWebApiExceptionLogger.cs) set up [here](App_Start/WebApiConfig.cs#L25) for Web API 2 controllers
  * See [this article](http://azure.microsoft.com/en-us/documentation/articles/app-insights-asp-net-exceptions/), for information on how you can collect unhandled exceptions from other application types 

####Worker Roles
* Add a TrackException(ex) call to report any exceptions you would like to collect, as shown [here](WorkerRoleA.cs#L109)

##Custom Events & Metrics
* These are very simple yet powerful API which let you track any event of significance, or a metric. You can add additional attributes to either type, that let you slice & dice the telemetry to get rich insights
* This worker role is instrumented to send a custom metric "SubscriberCount" as shown [here](WorkerRoleA.cs#L137).
* This worker role is instrumented to send 2 custom events as shown [here](WorkerRoleB.cs#L122) & [here](WorkerRoleB.cs#L187).
* **TBD** Links to articles that illustrate full power with custom attributes/dimensions etc.


##Traces
* The roles in this sample cloud service use System.Diagnostics traces. They are automatically collected with the [TraceListener](http://www.nuget.org/packages/Microsoft.ApplicationInsights.TraceListener) nuget added
* NLog, log4Net etc. also supported. See this [article](http://azure.microsoft.com/en-us/documentation/articles/app-insights-search-diagnostic-logs/) for more information on collecting traces from your worker roles

##Page View Telemetry
 * Collected automatically by adding the [JavaScript nuget](http://www.nuget.org/packages/Microsoft.ApplicationInsights.JavaScript)
 * You could also just add a JavaScript snippet to shared "master" file as shown [here](Views/Shared/_Layout.cshtml#L9)

##Performance Counters
* The following counters are collected by default:
  * \Process(??APP_WIN32_PROC??)\% Processor Time
  * \Memory\Available Bytes
  * \.NET CLR Exceptions(??APP_CLR_PROC??)\# of Exceps Thrown / sec
  * \Process(??APP_WIN32_PROC??)\Private Bytes
  * \Process(??APP_WIN32_PROC??)\IO Data Bytes/sec
  * \Processor(_Total)\% Processor Time
* For web roles, the following counters are collected in addition to the above:
  * \ASP.NET Applications(??APP_W3SVC_PROC??)\Requests/Sec	
  * \ASP.NET Applications(??APP_W3SVC_PROC??)\Request Execution Time
  * \ASP.NET Applications(??APP_W3SVC_PROC??)\Requests In Application Queue
* Moreover, you can specify additional custom or other windows performance counters as shown [here](ApplicationInsights.config#L14)

##Fine tune the Telemetry collection to work better for you
* If user/session telemetry is not applicable for your web/worker role, we recommend you remove the following telemetry modules and initializers from the ApplicationInsights.config file
  * [WebSessionTrackingTelemetryModule](ApplicationInsights.config#L34)
  * [WebUserTrackingTelemetryModule](ApplicationInsights.config#L35)
  * [WebSessionTelemetryInitializer](ApplicationInsights.config#L65)
  * [WebUserTelemetryInitializer](ApplicationInsights.config#L59)
* If your web/worker role has a mix of browser based clients & others, and you do have your web clients instrumented with the [JavaScript nuget](http://www.nuget.org/packages/Microsoft.ApplicationInsights.JavaScript):
  * Add SetCookie = false to the [WebSessionTrackingTelemetryModule](ApplicationInsights.config#L36) and [WebUserTrackingTelemetryModule](ApplicationInsights.config#L37) as mentioned [here](ApplicationInsights.config#L42)


## Next steps
* Set up availability monitors for your cloud service
* Set up alerts
* Set up continuous exports for the telemetry collected
* How to explore the ME/SE data

* [Monitor service health metrics](insights-how-to-customize-monitoring.md) to make sure your service is available and responsive.
* [Enable monitoring and diagnostics](insights-how-to-use-diagnostics.md) to collect detailed high-frequency metrics on your service.
* [Receive alert notifications](insights-receive-alert-notifications.md) whenever operational events happen or metrics cross a threshold.
* Use [Application Insights for JavaScript apps and web pages](app-insights-web-track-usage.md) to get client analytics about the browsers that visit a web page.
* [Monitor availability and responsiveness of any web page](app-insights-monitor-web-app-availability.md) with Application Insights so you can find out if your page is down.   