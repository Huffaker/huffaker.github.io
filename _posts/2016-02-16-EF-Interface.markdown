---
layout: post
title:  "Context Interface"
date:   2016-02-16
categories: blog
subtitle: "Setting Up an Entity Framework Context Interface"
image: "/images/EFInterface.png"
imagemin: "/images/EFInterface.min.png"
---

The standard template for database first entity framework builds out three types of files for use in your project. One of the scripted files, *.Context.cs, includes the basic methods you need to interact with the database. This is great for simple applications but if you intend to have a large audience use the application and would like to improve the testability of the application, it helps to make this context injectable. With this approach you can inject a context into your repositories and control the lifespan of the context to per request. This helps with security as well as improves the readability of your code.

Let's start by creating a new entity framework database first \*.edmx file. If you have already done this you can skip ahead a bit. Right click where you would like to place your edmx model and click Add -> new Item. Under "Visual C# Items" there should be an "ADO.NET Entity Data Model". I called mine "Entities" and after clicking add I entered my database connection and selected some tables to start with. If you haven't done this before, it's easy to go back and add more content from your database as you need. I now see several files all titled Entities.\*.

Since I don't know which version of entity framework you are using its very likely our line numbers will not match. Hopefully, I can at least get you close enough to see the change required. Let's start first by making a few changes to the *.Context.tt file. Starting from the top of your file, search down until you find the 'Using' statements (line 45 for me). Go ahead and add:

{% highlight javascript %}
using System.Collections.Generic;
{% endhighlight %}

Next find the class declaration line and add ", I<#=code.Escape(container)#>" to it like so:

{% highlight javascript %}
<#=Accessibility.ForType(container)#> partial class <#=code.Escape(container)#> : DbContext, I<#=code.Escape(container)#>
{% endhighlight %}

Next a slightly tricky part, we need to change the entities from DbSet to IDbSet which will enable us to interface them. You need to find the DbSet method in the *.Context.tt file (line 308 for me). Try searching for "DbSet" and it should show up. The new method is now:

{% highlight javascript %}
    public string DbSet(EntitySet entitySet)
    {
        return string.Format(
            CultureInfo.InvariantCulture,
            "{0} virtual IDbSet<{1}> {2} {{ get; set; }}",
            Accessibility.ForReadOnlyProperty(entitySet),
            _typeMapper.GetTypeName(entitySet.ElementType),
            _code.Escape(entitySet));
    }
{% endhighlight %}

The last change for the *.Context.tt is to change 'ObjectResult' to 'IEnumerable'. This is not as important of a change but there is no way to mock an ObjectResult, sort of a problem for unit testing. This was line 284 for me in the "FunctionMethod":

{% highlight javascript %}
return string.Format(
    CultureInfo.InvariantCulture,
    "{0} {1} {2}({3})",
    AccessibilityAndVirtual(Accessibility.ForMethod(edmFunction)),
    returnType == null ? "int" : "IEnumerable<" + _typeMapper.GetTypeName(returnType, modelNamespace) + ">",
    _code.Escape(edmFunction),
    paramList);
{% endhighlight %}

Alright, now we need to create the interface. Create a copy of the *.Context.tt file and rename to *.IContext.tt (in my case Entities.IContext.tt). You may have to do this by creating a new text template file and pasting in the code from the *.Context.tt file. Now let's change the class header to an interface (line 57 for me):

{% highlight javascript %}
<#=Accessibility.ForType(container)#> interface I<#=code.Escape(container)#> : IDisposable
{% endhighlight %}

Next up, remove the constructors (line 59 to 88). Basically, everything after the interface header down to the script where the foreach entitySet is located. Now remove the public virtual before each IDbSet declaration (Line 283). The updated DbSet method should look like this:

{% highlight javascript %}
    public string DbSet(EntitySet entitySet)
    {
        return string.Format(
            CultureInfo.InvariantCulture,
            "IDbSet<{1}> {2} {{ get; set; }}",
            Accessibility.ForReadOnlyProperty(entitySet),
            _typeMapper.GetTypeName(entitySet.ElementType),
            _code.Escape(entitySet));
    }
{% endhighlight %}

The final change to the interface file is to place two helper methods "SaveChanges" and "DbSet" back into the interface. These methods are hidden in the base class of the Context but we need to have access from the interface to accomplish much. I found a spot on line 73, just before the close of the interface definition.

{% highlight javascript %}
DbSet<T> Set<T>() where T : class;
int SaveChanges();
{% endhighlight %}

Alright, that should get you started. This interface will support tables, views, functions, procedures and all the basics that entity framework offers. There are certainly other features that may still need to be exposed in your interface depending on what you need for your project. To use dependency injection check out [Unity](https://github.com/unitycontainer/unity). Try testing your repositories with an in-memory version of entity framework using [Effort](https://github.com/tamasflamich/effort). I'll blog on this soon but having an interface is an important first step. 
