---
layout: post
title:  "Ajax Promises"
date:   2015-08-06  12:23:11
categories: blog
subtitle: "Promises on Multiple JQuery Ajax Calls"
image: "/images/mountain1.png"
---

I recently ran into a situation where I had multiple AJAX calls completing and needed to perform some logic when they ALL finish. I can never seem to remember how to do this when I need it so, here's a blog post, even if just for my benefit.

In my situation, I was simply hiding a load icon, and yes, I did not have a way to combine the AJAX calls which would have been more ideal performance wise. For this example we will be using JQuery for the AJAX calls and promise tooling.

What's a [promise][prom]? A very basic description in JavaScript, a promise is the ability to attach a function to the completion of one or multiple asynchronous events. In our case, we want to attach a function to the end of a series of AJAX request back to the server. JavasScript ECMAScript 2015 includes promises natively, but due to limited browser support and for added compatibility with JQuery AJAX calls, I recommend using JQuery, at least for now.

A JQuery AJAX call returns a promise. The basic work flow is to push each promise returned by an AJAX call into an array. We then iterate through the array firing each AJAX call and wait for them to complete. Once they ALL complete, we can kick off our final logic. Here's the code:

Define the array for holding all of our promises:

{% highlight ruby %}
  var deferreds = [];
{% endhighlight %}

Load some AJAX calls into our array:

{% highlight ruby %}
  for (var i = 0; i < 5; i++) {
    deferreds.push($.ajax({
    	  //Sample Data
    	  url: 'https://api.github.com/users/huffaker',
    	  type: 'GET'
      })
      .success(function(data) {
        console.log('Loaded Data');
      })
  	.error(function(error) {
    	  console.log(error);
  	})
    );
  }
{% endhighlight %}

Iterate through, running each AJAX call. When they all complete, run our completion logic:

{% highlight ruby %}
  $.when.apply($,deferreds).then(function() {
  	$('#state').text('Complete!');
  	console.log('All data has loaded');
  });
{% endhighlight %}

That's it, feel free to add comments below and let me know if you have any questions or cool insights on improving or expanding!

[Try the JSBin Here][JSBin]


[JSBin]: https://jsbin.com/lepusu/edit?html,console,output
[prom]: https://api.jquery.com/promise/