---
layout: post
title:  "Azure Mobile Services Insert"
date:   2015-08-24
categories: blog
subtitle: "Complex SQL Table Relationships with a JavaScript Backend"
image: "/images/azureInsert.png"
imagemin: "/images/azureInsert.min.png"
---

Azure mobile services provides a significant number of helpful tools for fast development and easy deployment. Excellent tooling is provided such as authentication, cross platform push notifications, and quick database development. However, all this framework can lead to confusion around performing more advanced concepts. In a series of posts I hope to cover some of my findings starting with managing complex database relationships.

In this quick demo, we will write an insert API endpoint within an azure mobile services JavaScript backend. If you haven't created a mobile services project yet, take a look at the [Microsoft blog](https://azure.microsoft.com/en-us/documentation/articles/mobile-services-windows-store-javascript-get-started-data/) which includes instructions for setting up a project. Come back when you're done for the more advanced table management.

To get started we need to first create two tables. Do this in the azure management / mobile services / data section, tables need to be structured in a way the mobile services is designed to understand or you will have "Internal Error" issues. Our first table is called "UserGroup". When this table is created it will receive several columns by default such as "id", "__createdAt", "__updatedAt", and "__version". We will add two more: "groupname" and "groupownerid", both as strings. Under the permissions tab, set insert to "Everyone" to allow for easy testing (Remember to change this back to authenticated users when you are done testing!) The second table is called "UserGroupUsers" which will receive two additional columns as well: "usergroupid" and "userid". Set the permissions to "Only Scripts and Admins" for all calls. This table should not be accessed outside of our table relationship.

![User Group Columns]({{ "/images/AMS-Insert/usergroupCol.png" | prepend: site.baseurl | prepend: site.url }})
![User Group Users Columns]({{ "/images/AMS-Insert/usergroupuserCol.png" | prepend: site.baseurl | prepend: site.url }})

The relationship we need to manage is that a UserGroup should contain a collection of UserGroupUsers. Using your favorite API test tool ([fiddler](http://www.telerik.com/fiddler" target="_blank">fiddler)) lets attempt to add a new UserGroup. Send a new JSON UserGroup using the [Mobile Service API](https://msdn.microsoft.com/en-us/library/azure/jj677200.aspx).

{% highlight javascript %}
  { "groupname": "Test Group",
   "groupownerid": "4eaa6c10-bdc4-4a2b-b7d9-883ba91d31cb" }
{% endhighlight %}

Checking the database directly or looking under the tab Browse should now include our new entry. Let's try again but this time with a collection of "UserGroupUsers" in our data object.

{% highlight javascript %}
  { "groupname": "Test Group 2",
   "groupownerid": "4a32a483-f852-4583-9684-02f09ea35cd7",
   "usergroupusers": [{"userid": "test user" }] }
{% endhighlight %}

This time you should recieve an error that you cannot save an object. Basically, you cannot save the collection using the default API script. We need to make a few adjustments to the insert script to manage this relationship.

{% highlight javascript %}
  function insert(item, user, request) {
    var usergroupusers = item.usergroupusers;
    delete item.usergroupusers;
    request.execute({
      success: function () {
        if (usergroupusers) {
          var userGroupUserTable = tables.getTable('UserGroupUser');
          usergroupusers.forEach(function (fieldUser) {
            userGroupUserTable.insert({
			 "usergroupid": item.id,
			 "userid": fieldUser.userid }, {
              error: function(err) {
                console.log(err);
              }
            });
          });
        }
        request.respond();
        },
          error: function (err) {
          request.respond();
        }
    });
  };
{% endhighlight %}

There are two steps we follow to make sure the save processes correctly. The first is, before the insert, to retrieve the "UserGroupUsers" and remove them from the item being saved. Next, execute the save and add the new values in the success call. Remember to call request.respond in the success handler to close the API insert call.

This same approach works for read, delete and update. The only thing to be aware of is that promises are required for the read call to return the full data item correctly. I plan on covering the read in a separate post so keep an eye out for that. Let me know in the comments if you have questions or any issues getting setup!
