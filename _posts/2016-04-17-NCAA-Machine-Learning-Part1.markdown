---
layout: post
title:  "NCAA Machine Learning Part 1"
date:   2016-04-17
categories: blog
subtitle: "Scraping NCAA Stats Data into a SQL Database"
image: "/images/NCAAMLPart1.png"
imagemin: "/images/NCAAMLPart1.min.png"
github: "/Huffaker/NCAA-Scraper"
---

Well the NCAA Tournament is now over and for most of us (me included) it could have gone a bit better. Want to step it up next year? This is part 1 of a two-part post on how to apply machine learning to NCAA game data. In this post I will walk through a short library I built for scraping stats into a SQL database. There should be enough information in the GitHub readme to get the project running but if you want to better understand what data you get and how it works, read on!

GitHub project found [here](https://github.com/Huffaker/NCAA-Scraper).

The application targets the NCAA stats site [http://stats.ncaa.org]( http://stats.ncaa.org) which stores data down to the player level. Of course, the framework is extendable if you wish to target a different site but as far as stats go I believe this to be the best. You can lookup how a specific player performed during a specific game which gives us a lot of flexibility when analyzing the data. We can aggregate the data of a player across a season, aggregate a team of players across a season or maybe look at a rolling performance (hint hint?). It certainly takes longer to pull this level of data (about 6 hrs a season) but you only have to do it once. It's not like it's going to change, just like our busted brackets aren't going to change (snap!).

So, the general workflow is this, we first pull the list of NCAA teams in the season (or seasons) we are targeting and save this list into SQL. Then for each team, we pull the list of players which means visiting a separate page for each. In this case, we pull stats like height, year in school, and other stats that we can't aggregate from the game data. Now for the really slow part, we pull each players game results. The NCAA stats site organizes this data by listing all the games a player was in and the stats for each game. This means a separate page for every player for every season. We then save all this into SQL and due to the number of data calls, we do this as we go incase anything crashes.

Before I dive into code, I want to discuss the method we use for scraping the web site. It's very possible to send out an HTTP request with C# and receive back the html file that contains our data. We can then parse this file for the data and call it a day. This approach will even run significantly faster. The problem is in the parse step. Writing decent and predictable regex to find the fields we need is not a simple task and we can easily pull text when we expect integers and end up with garbage data. Having done a fair amount of web development, I opted for the slower but saner approach of parsing the html as a web site.

Using a hidden web browser tool [PhantomJS](http://phantomjs.org/) we can navigate to the target web site and load the html as a DOM (Document Object Model). Now I can use all the available web technology to pares the file. Since the NCAA site already loads JQuery, I went ahead and used this to search for the data I need. Another benefit, if the NCAA changes its website next year, I can update the JQuery code much faster than I could Regex. Unfortunately, websites changing is part of the fun of screen scraping.

Alright, let's look at some code! The main logic is in the abstract **Scraper** object which handles connecting to the target web site and pulling data. Each data type we are pulling in (teams, players, games) is then collected with an object that inherits from Scraper. The Scraper object only has one public method and a few protected fields and methods:

{% highlight javascript %}
protected int totalCalls = 0;
protected string scrapResult { get; set; }
protected string javascriptCode { get; set; }
public void RunScrap(string url)
protected abstract void ProcessResult(string url);
protected void CleanUpBrowser();
protected void LogResult(string url, int results);
{% endhighlight %}

The totalCalls field is a counter used to display the status as the application runs.  The scrapResult is a string field that receives a JSON string of the data the Scraper collects. In this application, we use [JQuery](https://jquery.com/) to retrieve the data from the web site we are targeting. This JQuery JavaScript code script is placed in the javascriptCode field for use by the Scraper prior to calling RunScript. In most cases, we run RunScript multiple times as we iterate through each team or each player. Since each of these requires a different web url, we pass this in here.

Let's look a little closer at the RunScript method. First we kick up a PhantomJS browser if we haven't already. We then navigate this browser to the target website.  Once this browser fully loads, we can execute the JQuery code to get the data back as a JSON string. We then pass this down to the ProcessResults method which is overridden by the inheriting object. It looks like this:

{% highlight javascript %}
public void RunScrap(string url)
{
  // Create an instance of Phantom browser 
    if (browser == null)
        browser = new PhantomJS();
    scrapResult = AquireText(url, javascriptCode);
    ProcessResult(url);
}

private string AquireText(string strURL, string javascriptCode)
{
	using (var ms = new MemoryStream())
	{
		browser.RunScript(@"
			var system = require('system');
			var page = require('webpage').create();
			page.open('" + strURL + @"', function(status) {
			if(status === 'success') {
				var data = page.evaluate(function() {
					" + javascriptCode + @"
				});
				system.stdout.writeLine('!~!' + data + '!~!');
			}
			phantom.exit();
		});"
		, null, null, ms);
		ms.Position = 0;
		var sr = new StreamReader(ms);
		var result = sr.ReadToEnd();
		result = result.Split(new [] { "!~!"}, StringSplitOptions.None)[1];
		return result;
	}
}
{% endhighlight %}

One thing to note, the result from PhantomJS returns a bunch of header information along with the data. To help with this we add the text string "!~!" which we use to find just the data piece. To process the JSON data we can use the library Newtonsoft.Json. For example, to de-serialize game data we pass in the object type were attempting to cast too:

{% highlight javascript %}
var result = JsonConvert.DeserializeObject<List<GameModel>>(scrapResult);
{% endhighlight %}

The final step is to save the data into SQL. The .Net library gives us a great library for this purpose called System.Data.SqlClient which has the SqlBulkCopy class. Open a new connection, set the target table and then write the object to the database. The only tricky piece is SqlBulkCopy can't write any object, it has to be a DataTable type. A super helpful method for doing this using refection for any object is the ToDataTable method.

{% highlight javascript %}
public static DataTable ToDataTable<T>(List<T> items)
{
	var dataTable = new DataTable(typeof(T).Name);

	//Get all the properties
	var props = typeof(T).GetProperties(BindingFlags.Public | BindingFlags.Instance);
	foreach (var prop in props)
	{
			//Setting column names as Property names
			dataTable.Columns.Add(prop.Name, Nullable.GetUnderlyingType(prop.PropertyType) ?? prop.PropertyType);
	}
	foreach (var item in items)
	{
		var values = new object[props.Length];
		for (var i = 0; i<props.Length; i++)
		{
				//inserting property values to datatable rows
				values[i] = props[i].GetValue(item, null);
		}
		dataTable.Rows.Add(values);
	}
	//put a breakpoint here and check datatable
	return dataTable;
}
{% endhighlight %}

Now the process of loading the data is very simple. Below is the code but if your following along in the GitHub project you will notice it's slightly different. In the full project I am actually clearing a staging table, filling it with data and then merging it into the main data table. This lets me bring in partial data sets without worrying about duplicate data. Something to keep in mind but not always necessary.

{% highlight javascript %}
public static void LoadData<T>(IEnumerable<T> data, string table, string merge, string connectionString)
{
	using (var connection = new SqlConnection(connectionString))
	{
		connection.Open();
		using (var bulkCopy = new SqlBulkCopy(				 
						connection,
							SqlBulkCopyOptions.TableLock |
							SqlBulkCopyOptions.FireTriggers |
							SqlBulkCopyOptions.UseInternalTransaction,
							null))
		{
		    bulkCopy.DestinationTableName = table;

		    //Load in new data
		    bulkCopy.WriteToServer(ToDataTable(data.ToList()));

			connection.Close();
		}
	}
}
{% endhighlight %}

Alright, now our data is scraped from our target website and loaded into a SQL database! In my next post we will look at applying machine learning to the data to predict game results.

 I'll wrap up this post by just reminding you that screen scraping can get you in trouble, especially if you hit a server too often and hurt the performance. There are a lot of great discussions out there on this topic but nothing definitive on the frequency you can hit a website site. This project plays it fairly safe by waiting for a full page load before continuing but, sorry, I am not responsible if you get sued even with the code as is!
