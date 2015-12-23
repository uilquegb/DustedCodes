﻿<!--
    Published: 2015-04-06 22:29
    Author: Dustin Moris Gorski
    Title: Demystifying ASP.NET MVC 5 Error Pages and Error Logging
    Tags: asp-net mvc error-pages error-logging
-->
<p>Custom error pages and proper error logging are two elementary and yet very confusing topics in ASP.NET MVC.</p>
<p>There are numerous ways of implementing error pages in ASP.NET MVC 5 and when you search for advice you will find a dozen different StackOverflow threads, each suggesting a different implementation.</p>
<h2>Overview</h2>
<h3>What is the goal?</h3>
<p>Generally when speaking of error pages and error logging I mean error pages and logging for <strong>unhandled exceptions</strong> in your application.</p>
<p>The basic goal is:</p>
<ol>
    <li>
        Human friendly error pages
        <ul>
            <li>Custom page per error code (e.g.: 404, 403, 500, etc.)</li>
            <li>Preserving the HTTP error code in the response to avoid search engines index error pages</li>
        </ul>
    </li>
    <li>Global error logging for unhandled exceptions</li>
</ol>
<h3>Error pages and logging in ASP.NET MVC 5</h3>
<p>I am sure one could think of many more possible solutions to the problem, but typically you will find implementations which involve at least one or a combination of many of these methods:</p>
<ul>
    <li><a href="https://msdn.microsoft.com/en-us/library/system.web.mvc.handleerrorattribute%28v=vs.118%29.aspx">HandleErrorAttribute</a></li>
    <li><a href="https://msdn.microsoft.com/en-us/library/system.web.mvc.controller.onexception%28v=vs.118%29.aspx">Controller.OnException Method</a></li>
    <li><a href="https://msdn.microsoft.com/en-us/library/fwzzh56s(v=vs.140).aspx">Application_Error event</a></li>
    <li><a href="https://msdn.microsoft.com/en-us/library/h0hfz6fc%28v=vs.85%29.aspx">customErrors element</a> in web.config</li>
    <li><a href="https://msdn.microsoft.com/en-us/library/ms690497%28v=vs.90%29.aspx">httpErrors element</a> in web.config</li>
    <li>Custom <a href="https://msdn.microsoft.com/en-us/library/ms178468%28v=vs.85%29.aspx">HttpModule</a></li>
</ul>
<p>That's a lot of different ways for handling an error and they all have a justifyable use case. There is no golden solution which works for every application, so it is good to know how they work and when they are best applied.</p>
<p>Before I will go through each approach in more detail I want to explain some fundamentals which will hopefully make the topic a lot easier to understand.</p>
<h3>ASP.NET MVC Fundamentals</h3>
<p>ASP.NET MVC is nothing more than an <a href="https://msdn.microsoft.com/en-us/library/ms227675%28v=vs.100%29.aspx">HttpHandler</a> plugged into the ASP.NET framework. The easiest way to illustrate this is by opening the Global.asax.cs:</p>
<pre><code>public class MvcApplication : System.Web.HttpApplication</code></pre>
<p>Navigating to the implementation of HttpApplication will reveal the underlying IHttpHandler and IHttpAsyncHandler interfaces:</p>
<pre><code>public class HttpApplication : IComponent, IDisposable, IHttpAsyncHandler, IHttpHandler</code></pre>
<p>ASP.NET itself is a larger framework to process incoming requests. Even though it could handle incoming requests from different sources, it is almost exclusively used with <abbr title="Internet Information Services">IIS</abbr>. It can be extended with <a href="https://msdn.microsoft.com/en-us/library/bb398986%28v=vs.140%29.aspx">HttpModules and HttpHandlers</a>.</p>
<p>HttpModules are plugged into the pipeline to process a request at any point of the <a href="https://msdn.microsoft.com/en-us/library/ms178473(v=vs.85).aspx">ASP.NET life cycle</a>. A HttpHandler is responsible for producing a response/output for a request.</p>
<p>IIS (Microsoft's web server technology) will create an incoming request for ASP.NET, which then will start processing it and eventually initialize the HttpApplication (the default handler) and create a response:</p>
<a href="https://www.flickr.com/photos/130657798@N05/16862010839" title="IIS, ASP.NET and MVC architecture by Dustin Moris Gorski, on Flickr"><img src="https://farm9.staticflickr.com/8736/16862010839_64d17c3268_o.gif" alt="IIS, ASP.NET and MVC architecture"></a>
<p>The point is that ASP.NET can only handle requests which IIS has forwarded to it. This is determined by the registered HttpHandlers (e.g. by default a request to a .htm file is not handled by ASP.NET).</p>
<p>And finally, MVC is only one of potentially many registered handlers within ASP.NET.</p>
<p>This is crucial to understand the different solutions for error handling.</p>
<h2>Breaking down the options</h2>
<h3>HandleErrorAttribute</h3>
<p>The HandleErrorAttribute is an MVC FilterAttribute, which can be applied to a class or method:</p>
<pre><code>namespace System.Web.Mvc
{
    [AttributeUsage(
        AttributeTargets.Class | AttributeTargets.Method, 
        Inherited = true, 
        AllowMultiple = true)]
    public class HandleErrorAttribute : FilterAttribute, IExceptionFilter
    {
        // ...
    }
}</code></pre>
<p>It's error handling capabilities are limited to action methods within the MVC framework. This means it won't be able to catch and process exceptions raised from outside the ASP.NET MVC handler (e.g. exceptions at an earlier stage in the life cycle, errors in other handlers, etc.) nor any exceptoins where your action method is not part of the call stack (e.g. routing errors, etc.).</p>
<p>The HandleErrorAttribute only handles 500 internal errors. For example this will not be caught by the attribute:</p>
<pre><code>[HandleError]
public ActionResult Index()
{
    throw new HttpException(404, "Not found");
}</code></pre>
<p>You can use the attribute to decorate a controller class or a particular action method. It supports custom error pages per exception type out of the box:</p>
<pre><code>[HandleError(ExceptionType = typeof(SqlException), View = "DatabaseError")]]</code></pre>
<p>In order to get the HandleErrorAttribute working you also need to turn on customErrors in your web.config:</p>
<pre><code>&lt;customErrors mode="On" /&gt;</code></pre>
<h4>Use case</h4>
<p>The HandleErrorAttribute is the most limited in scope. Many application errors will bypass it and therefore it is not ideal for generic cross application error handling.</p>
<p>It is a great tool for action specific error handling though (e.g. additional fault tolerance for critical action methods).</p>
<h3>Controller.OnException Method</h3>
<p>This method gets called if any action method inside the controller throws an exception. Unlike the HandleErrorAttribute it will also catch 404 and other HTTP error codes and doesn't require setting customErrors mode on.</p>
<p>The implementation is simple, just override the OnException method in your controller:</p>
<pre><code>protected override void OnException(ExceptionContext filterContext)
{
    filterContext.ExceptionHandled = true;
            
    // Redirect on error:
    filterContext.Result = RedirectToAction("Index", "Error");
    // OR set the result without redirection:
    filterContext.Result = new ViewResult
    {
        ViewName = "~/Views/Error/Index.cshtml"
    };
}</code></pre>
<p>You can check if the exception has alrady been handled at an earlier stage (e.g.: the HandleErrorAttribute):</p>
<pre><code>if (filterContext.ExceptionHandled)
    return;</code></pre>
<p>Many solutions on the internet suggest to create a base controller class and implement the method in one location for a generic error handler.</p>
<p>However, the fact that it is almost as limited as the HandleErrorAttribute, it is not an ideal solution for generic error handling. You will end up duplicating your work at least in one other place.</p>
<h4>Use case</h4>
<p>It gives a little bit more flexibility than the HandleErrorAttribute, but it is still too limited for generic error processing. It seems to be popular when you need to distinguish your error handling between regular and AJAX requests on a controller level.</p>
<h3>Application_Error event</h3>
<p>The Applicatoin_Error method is far more generic than the previous two options. It is not limited to the MVC scope any longer and needs to be implemented in the Global.asax.cs:</p>
<pre><code>protected void Application_Error(Object sender, EventArgs e)
{
    var raisedException = Server.GetLastError();
    // Process exception
}</code></pre>
<p>If you've noticed it doesn't come from an interface, an abstract class or an overriden method. It is purely convention based, similar like the Page_Load event in ASP.NET Web Forms applications.</p>
<p>Any unhandeled exception within ASP.NET will bubble up to this event. There is also no concept of routes anymore (because it is outside the MVC scope). If you want to redirect to a specific error page you have to know the exact URL or configure it to co-exist with customErrors or httpErrors.</p>
<h4>Use case</h4>
<p>In terms of generic error logging this is a great place to start with! It will capture all errors which haven't been handled at an earlier stage. But be careful, if you have used controller exception handling and set <code>filterContext.ExceptionHandled = true</code> then the exception will not bubble up to Applicatoin_Error.</p>
<p>For custom error pages it is still not perfect. This event will trigger for all ASP.NET errors, but what if someone navigates to a URL which isn't handled by ASP.NET? For example try navigating to http://{your-applicatoin}/a/b/c/d/e/f/g/h. The route is not mapped to ASP.NET and therefore will not be captured in your application and provided with a custom error page from the Application_Error event.</p>
<h3>customErrors in web.config</h3>
<p>This web.config setting enables you to provide a default error page as well as custom error pages for specific error codes:</p>
<pre><code>&lt;system.web&gt;
    &lt;customErrors mode="On" defaultRedirect="~/Error/Index"&gt;
        &lt;error statusCode="404" redirect="~/Error/NotFound"/&gt;
        &lt;error statusCode="403" redirect="~/Error/BadRequest"/&gt;
    &lt;/customErrors&gt;
&lt;system.web/&gt;
</code></pre>
<p>The default implementation redirects the user to the specified error page. This is really bad practise because it will change the original HTTP error code to 302 HTTP Redirect and eventually finish with HTTP 200 OK at the error page. Additionally the original URL will have changed as well. This is not only confusing but has other negative side effects too, like Google will start indexing your error pages!</p>
<p>Luckily you can change this behaviour by setting the redirectMode to ResponseRewrite:</p>
<pre><code>&lt;customErrors mode="On" redirectMode="ResponseRewrite"&gt;</code></pre>
<p>This fixes the initial problem, but now you will end up with an error when redirecting to your error page:</p>
<blockquote class="yellow-screen-of-death">
    <header>Runtime Error</header>
    <p>An exception occurred while processing your request. Additionally, another exception occurred while executing the custom error page for the first exception. The request has been terminated.</p>
</blockquote>
<p>It is because ResponseRewrite mode uses <a href="https://msdn.microsoft.com/en-us/library/ms525800%28v=vs.90%29.aspx">Server.Transfer</a> under the covers, which looks for a file on your file system. As a result we need to change the redirect path to a static file, for example to an .aspx or .html file:</p>
<pre><code>&lt;customErrors mode="On" redirectMode="ResponseRewrite" defaultRedirect="~/Error.aspx"/&gt;</code></pre>
<p>Now there is only one last issue left with this configuration: The response code of the error page will still be 200 OK. The only fix for this is to manually set the correct error code in your .aspx error page:</p>
<pre><code><% Response.StatusCode = 404; %></code></pre>
<p>This is already pretty good in terms of custom error pages, but we can do better! Noticed how the customErrors section goes into the system.web section? This means we are still in the scope of ASP.NET.</p>
<p>Files and routes which are not handled by your ASP.NET application will render a default 404 page from IIS (e.g.: http://myapp/some/path/not/existing/image.gif).</p>
<p>Another downside of customErrors is that if you use a <a href="https://msdn.microsoft.com/en-us/library/system.web.mvc.httpstatuscoderesult%28v=vs.118%29.aspx">HttpStatusCodeResult</a> instead of throwing an actual exception then it will bypass the ASP.NET customErrors mode and get handled on an IIS level again:</p>
<pre><code>public ActionResult Index()
{
    return HttpNotFound();
    //throw new HttpException(404, "Not found");
}</code></pre>
<p>In this case there is no hack which we can apply to display a friendly error page which comes from customErrors.</p>
<h4>Use case</h4>
<p>The customErrors setting gets us very far, but still has its limits. You can think of it as a legacy version of httpErrors, which has been introduced with IIS 7.0.</p>
<p>The only time when customErrors still makes sense is if you can't use httpErrors, because you are running on IIS 6.0 or lower.</p>
<h3>httpErrors in web.config</h3>
<p>The httpErrors section is similar to customErrors, but with the main difference that it is an IIS level setting rather than ASP.NET and therefore needs to go into the system.webserver section in the web.config:</p>
<pre><code>&lt;system.webServer&gt;
    &lt;httpErrors errorMode="Custom" existingResponse="Replace"&gt;
      &lt;clear/&gt;
      &lt;error statusCode="404" path="/WebForms/Index.aspx" responseMode="ExecuteURL"/&gt;
    &lt;/httpErrors&gt;
&lt;system.webServer/&gt;</code></pre>
<p>It allows more configuration than customErrors but has its own little caveats. I'll try to explain the most important settings in a nutshell:</p>
<ul>
    <li>httpErrors can be inherited from a higher level (e.g. set in the machine.config)</li>
    <li>Use the <code>&lt;remove/&gt;</code> tag to remove an inherited setting for a specific error code.</li>
    <li>Use the <code>&lt;clear/&gt;</code> tag to remove all inherited settings.</li>
    <li>Use the <code>&lt;error/&gt;</code> tag to configure the behaviour for one error code.</li>
    <li>
        responseMode &quot;ExecuteURL&quot; will render a dynamic page with status code 200.
        <ul>
            <li>The workaround to set the correct error code in the .aspx page works here as well.</li>
        </ul>
    </li>
    <li>responseMode &quot;Redirect&quot; will redirect (302) to any URL.</li>
    <li>
        responseMode &quot;File&quot; will preserve the original error code and output the static file.
        <ul>
            <li>.aspx files will get output in plain text.</li>
            <li>.html files will render as expected.</li>
        </ul>
    </li>
</ul>
<p>The main advantage of httpErrors is that it is handled on an IIS level. It will literally pick up all error codes and redirect to a friendly error page. If you want to benefit from master pages I recommend to go with the ExecuteURL and status code fix appraoch. If you want to have rock solid error pages which IIS can serve even when everything else burns, then I'd recommend to go with the static file approach (preferably .html files).</p>
<h4>Use case</h4>
<p>This is currently the best place to configure friendly error pages in one location and to catch them all. The only reason not to use httpErrors is if you are still running on an older version of IIS (< 7.0).</p>
<h3>Custom HttpModule</h3>
<p>Last but not least I wanted to quickly touch on a custom HttpModule. It has nothing to do with friendly error pages (httpErrors is the way to go), but it is a great tool for generic error logging.</p>
<p>The error event which can be subscribed to from insde a custom HttpModule behaves the exact same way as the Application_Error event. If you have both implemented then it gets called before Applicatoin_Error.</p>
<p>The benefit of the HttpModule is that it is reusable in other ASP.NET applications. Adding/Removing an HttpModule is as simple as adding or removing one line in your web.config:</p>
<pre><code>&lt;system.webServer&gt;
&lt;modules&gt;
    &lt;add name="CustomModule" type="SampleApp.CustomModule, SampleApp"/&gt;
&lt;/modules&gt;
&lt;/system.webServer&gt;</code></pre>
<p>In fact someone has already created a powerful and reusable error logging module and it is open source and called <a href="https://code.google.com/p/elmah/">ELMAH</a>.</p>
<p>If you need to create application wide error logging, I highly recommend to look at this project!</p>
<h2>Final words</h2>
<p>I hope this overview was helpful to better understand which approach might fit best your personal error handling requirements in an ASP.NET MVC project.</p>
<p>Each of the techniques has a use case and it really depends on what you want to do.</p>