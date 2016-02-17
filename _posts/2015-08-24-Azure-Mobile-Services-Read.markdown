---
layout: post
title:  "Azure Mobile Services Read"
date:   2015-09-09
categories: blog
subtitle: "Complex SQL Table Relationships with a JavaScript Backend"
image: "/images/azureRead.png"
imagemin: "/images/azureRead.min.png"
---


Let's take another look at managing SQL complex relationships in an Azure mobile services with a node.js JavaScript backend. This time, let's build on the previous post and do a read operation to pull database entities that contain a collection of member entities. In case you missed my previous post for the insert operation take a look [here](http://jameshuffaker.com/#!/blog/azure-mobile-services-insert).

Azure mobile services is excellent for standing up a backend quickly with excellent tooling for mobile applications (or web applications, why not every platform possible right?).  However, when working with node.js, you are almost always forced into a no SQL approach. In most cases, sure, that's a great solution but SQL still has plenty of advantages including hosting costs and existing team skill sets. Maybe your data is in SQL and needs to stay there for your existing applications? If you're looking to use node.js and SQL then azure mobile services is certainly an option. Beware, there are certainly some limits such as table schema requirements and the tooling doesn't come even close to the capabilities of database first entity framework (pretty much on par with code first however).

Let's quickly frame the problem we are looking to solve. We have a group entity in our database called "UserGroup" that contains a collection of "UserGroupUsers" that we created in the previous [post](http://jameshuffaker.com/#!/blog/azure-mobile-services-insert). The default read operation in azure mobile services will return all the "UserGroups" in the database but even with a foreign key in place, the services will not return the list of "UserGroupUsers" by default. We need to add in some additional functionality to the read operation to pull these collections of users and add them to our returned data object.

Alright, let's dive in. If you haven't already done so clone the azure mobile service project into Visual Studio using the git link available in the configuration screen of the azure management portal. Next, (again, if you haven't already) install node.js [here](https://nodejs.org/). All set? Open a command prompt, navigate to the services folder in your project folder and install q.

{% highlight javascript %}
npm install q
{% endhighlight %}

[q](https://github.com/kriskowal/q) is a JavaScript promises library for node.js. Timing is extra important for the read operation, promises will greatly simplify this process. Now, we can add the following require statement to the top of our read operation:

{% highlight javascript %}
var q = require('q');
{% endhighlight %}

Now, open the UserGroup read operation. If it doesn't exist, go ahead and add a new file titled "usergroup.read.js" to the "tables" folder. The read operation takes three parameters: A query containing the OData filters that have been passed in, the user with authentication details if available, and a request object. We are going to start by executing the request to pull the data for the parent table. The success handler option for this call returns the data that will be passed as json back to the client. Anything you add to the data object will be returned to the user which gives us our window to make several additional calls to the database to get the collection entities. Since database calls are asynchronous, we need to use promises to wait for all the calls to complete before we return back to the client. Below is the basic workflow:

{% highlight javascript %}
function read(query, user, request) {
  
  request.execute({
    
    success: function (data) {
      //Add UserGroupUsers
      var allPromise = [];
      var userGroupUserTable = tables.getTable('UserGroupUser');
      
      data.forEach(function (dataItem) {
        allPromise.push(
          loadUserGroupUsers(dataItem, userGroupUserTable)
        );
      });
      q.all(allPromise)
        .then(function () {
           request.respond();
         }
       );
    },
    error: function (err) {
      request.respond();
    }
  });
}
{% endhighlight %}

The method loadUserGroupUsers will need to first create a promise that tracks the progress of the sub call. Second, we need to start the asynchronous call to grab all the collection entities and add them to our data object. Finally, return the promise.

{% highlight javascript %}
function loadUserGroupUsers(dataItem, userGroupUserTable) {
  var deferred = q.defer();
  userGroupUserTable.where({
    usergroupid: dataItem.id,
    __deleted: 0
  }).read({
    success: function (existingItems) {
      dataItem.usergroupusers = existingItems;
      deferred.resolve();
    },
    error: function (err) {
      //In my case we are going to
      //log the error but continue
      console.log(err);
      deferred.resolve();
    }
  });
  return deferred.promise;
}
{% endhighlight %}

A few things to notice in the loadUserGroupUsers function. We are filtering to pull the items in the UserGroupUsers table that have the id of our current parent object and checking that delete flag. There is an option to turn off the delete flag in the Azure portal and just delete rows instead. Next, in my case I have decided to just log any read errors and continue. You may need to approach this differently depending on your application needs. Also, it's important to mention that even though we are only reading UserGroupUsers through the UserGroup read endpoint, this table still needs to be created within the azure mobile services portal. This ensures the table has the correct schema including a guid id column, delete flag, created at, etc...

That should be it, when all the asynchronous calls have completed, the request is closed with the request.respond() call. The data will be sent back as json to the client. Feel free to reach out in the comments with questions/issues and I'll do what I can to help!