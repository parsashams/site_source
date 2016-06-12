---
layout: post
title: "Quick Data Simulation for Prototyping"
date: 2014-11-12 04:00:55 -0500
categories: blog data
---
<p><img class="alignright wp-image-63 " src="{{ site.baseurl }}/assets/befunky_crash-test-dummy-man1.jpg?w=300" alt="" width="300" height="201" />Few people actually admit it, but dummy data plays a crucial role in BI applications/dashboards! Why? Because usually the UI team is waiting for the data development team to get the data ready before they can move forward. The best way to avoid this idle time is to generate some made-up (or dummy) data and use that to populate the database. But good dummy data is surprisingly difficult to generate.</p>
<p>How do we define good dummy data? Simply three conditions:</p>
<ol>
<li><strong>It must have referential integrity: </strong>No broken links</li>
<li><strong>It must have proper features to allow for unit testing of the program logic:</strong> a data-set that is full of "test"s, "0"s and "1"s is not really useful for testing purposes because you would have no way of testing if you data is getting aggregated in the UI layer.</li>
<li><strong>It must make sense:</strong> test12345 is not an acceptable male name. (<em>Well, this one is a nice to have. Your dummy data will still work but the presentation will probably not knock your stakeholder's socks off).</em></li>
</ol>
<p>How to make such data. Here is an easy two step approach:</p>
<ul>
<li><strong>Step 1: Populate lookup table elements with data that is as close to real data as possible.</strong></li>
<li><strong>Step 2: Populate fact tables, using joins and random functions to your advantage. </strong></li>
</ul>
<p><!--more--></p>
<h3><strong>Step 1: Populate lookup table elements with data that is as close to real data as possible.</strong></h3>
<p>The complexity of this step can vary based on the complexity of your data model. The fist step in this process is always the same, you need to start at the outer edges of the data model. The outer tables are those with only one primary key and no foreign keys. You can write a simple queries on the database metadata to extract the list of tables sorted by the number of key constraints. Here is a sample script to do this in postgresql:</p>
<p>[code lang="sql"]<br />
SELECT tables.table_name,<br />
       columns.column_name,<br />
       columns.data_type,<br />
       columns.ordinal_position,<br />
       CASE WHEN position('dk' in columns.column_name) &gt; 0 THEN 1 END AS column_type<br />
FROM information_schema.tables,<br />
     information_schema.columns<br />
WHERE tables.table_name = columns.table_name<br />
  AND tables.table_schema = columns.table_schema<br />
  AND tables.table_schema = 'a'<br />
ORDER BY<br />
  tables.table_name DESC,<br />
  columns.ordinal_position ASC<br />
[/code]</p>
<p>Now In our data model all the key elements had a "_dk" postfix, and that is how I identify the keys. <em>(I am sure there are better ways to do this, but this is my quick and dirty way!)</em>. In any case, this Query will give us a list of lookup table names ordered by the number of keys. To populate the tables we start with the ones with only one primary key and no foreign keys. Now, we can populate each table manually or we can generate random data. The best way to do this for lookup tables is to use a free online service designed for this purpose (Remember, this is just dummy data, with a shelf-life of maybe a couple of months, so no need to over engineer this). Here are two good tools:</p>
<ul>
<li><a href="http://www.generatedata.com/">http://www.generatedata.com/</a></li>
<li><a href="http://dummydata.me/">http://dummydata.me/</a></li>
</ul>
<p>Both of these tools allow for some flexibility and have the most common types of data like phone number, dates, names, etc. So you can They are very easy to work with and allow you to create clean dummy lookup tables quickly.</p>
<p>Now, let's look at an extreme example. Let's say we want to generate really realistic dummy data. For example, we want to populate the 'ad dimension' table for an online advertising company, and we want to have ads from various industries. If we generate this data using simple random function, the resulting data will be very uniform, since we are using a uniform random distribution function. To solve this, we need to tweak the probability distribution a little bit, via a little bit of code. This allows us to define our own probability distribution and impose conditions on the lookup table elements. Here's the example of the advertising company using R:</p>
<p>[code lang="R"]<br />
# Ad Dimension Table ------------------------------------------------------<br />
ncampaigns &lt;- 30</p>
<p>adID &lt;- 1:ncampaigns</p>
<p>Industries &lt;- c(&quot;Health Care&quot;, &quot;Automotive&quot;, &quot;Fast Food&quot;, &quot;Sporting Goods&quot;, &quot;Software&quot;)<br />
distInd     &lt;- c(1,5,2,10,3)<br />
adIndustry &lt;- sapply(adID, function(ind) { sample(Industries, size=1, prob=distInd) })<br />
[/code]</p>
<p>In the code above we are using a weighted vector to impose a probability distribution on the campaigns so that the resulting data has our desired composition. R is very powerful with this type of vector manipulation, so we can do all that in only three lines. Luckily we do not usually have to resort to simulations like this with dummy data generation.</p>
<p>Now that we have all the lookup tables populated, it is time to populate the fact tables. This is actually quite easy.</p>
<h3><strong>Step 2: Load Fact Tables Using Joins and Random Functions<br />
</strong></h3>
<p>There are two methods to simulating dummy data for fact tables.</p>
<p>First, we can use use bulk insert statements in SQL by utilizing random functions that are native to SQL to generate the desired fact values. I use the random() function extensively and then use a variation of different functions on top of them to create the desired effect. The <tt>random()</tt> function in SQL generates a random number between zero and one using the uniform distribution. This might sound too simplistic, but it covers the majority of dummy data use cases. We can use variations of the random function along with simple math to create the desired fact values. For example, to create an integer value between 0 and 10 we can write</p>
<p>[code lang="sql" gutter="false"] floor(random()*11) [/code]</p>
<p>We can also get creative and create dependencies between the facts and levels of our dimension attributes. Say we want to generate a fact called Status that is to vary based on two or more other attribute, we can use the remainder operator <tt>(%)</tt> to generate a dependent value.:</p>
<p>[code lang="sql" gutter="false"] (a.FactCalcTableKey+b.FinYearKey) % 3 [/code]</p>
<p>In this case we have three possible values for status.</p>
<p>Sky is really the limit with this technique. You can play with various functional forms to generate the data in the format that is needed. Here is a more extensive example code:</p>
<p>[code lang="sql"]</p>
<p>SET SCHEMA 'public';<br />
TRUNCATE TABLE public.fact_calculation;<br />
SET SEED = 0.42;<br />
INSERT INTO public.fact_calculation<br />
(  <br />
 SELECT  <br />
  a.FactCalcTableKey AS FactCalcTableKey,<br />
  b.FinYearKey AS FinYearKey,<br />
  c.TimePeriodKey AS TimePeriodKey,<br />
  ( (a.FactCalcTableKey+b.FinYearKey) % 3 ) +1 AS StatusKey,<br />
  floor(random()*11) AS Score,<br />
  (CASE WHEN random() &gt; 0.2 THEN 'High' ELSE 'low' END) AS Rate,<br />
  5 AS Baseline,<br />
  floor(random()*10) AS Score1,<br />
  floor(random()*11) AS Score2,<br />
  floor(random()*20) AS Score3,<br />
  floor(random()*1000) AS FinImpact,<br />
  floor(random()*100) AS CountEvents,<br />
  random()*3 AS Variance<br />
FROM<br />
  lu_OrgStructure a,<br />
  lu_FiscYear b,<br />
  lu_TimePeriod c<br />
WHERE<br />
  c.TimePeriodKey IN (1,2,3,4)<br />
GROUP BY<br />
 FactCalcKey, FinYearKey, TimePeriodKey<br />
 LIMIT 1000<br />
); </p>
<p>[/code]</p>
<p>This amount of data should be enough to allow developers to move forward with their work, but again, if we have an extreme case that requires very sophisticated simulated dummy data, we can use a higher level language to program in any relationships that could exist. Let me give you an example. Remember the online advertising example, imagine the outcome variable in that case was the decision of the user to click on the ad or not. This would be a binary variable, but creating that as a simple <tt>random()</tt> would make up for some very lackluster dashboards because everything would be uniform and provide no features for the user to explore. But what if we used R to create a logistic function to simulate the users response based on a bunch of probability estimates? That way we can work a number of arbitrary assumptions into the model, say if we wanted adult male users to have a more positive response to ads involving sports, we could do that. Following code should give you a good idea how to this (confluence does not support syntax highlighting for <tt>R</tt>). Here, we are using R to simulate a binary outcome variable with an arbitrary probability function using a logistic link function. This is just a fancier way of doing the same thing we were doing with SQL <tt>random()</tt> function. The only complexity is that we are using a logistic function which gives us a probability value given a relatively complex setup of independent variables such as Gender, Age and Day of Week. We then use this simulated probability value to forecast whether a click event is observed. See the code below:</p>
<p>[code lang="r"]<br />
e = rnorm(k) # K is the number of rows in the fact table<br />
logit = function(x){ exp(x)/(1+exp(x))}<br />
p=logit(-  1 * (gender==&quot;F&quot;)<br />
        + .1 * (gender==&quot;M&quot;)<br />
        + .2 * (adIndustry==&quot;Tech&quot;)<br />
        + .3 * (abs(age-35))<br />
        + .3 * (weekday == &quot;Friday&quot; )<br />
        + .5 * (toString(progIndustry) == toString(adIndustry))<br />
        -12 + e)</p>
<p>pairs(data.frame(p, as.numeric(hourID), dateID-joindateID), col=gender)<br />
hist(p, prob=TRUE)<br />
mean(p)<br />
#generate the click using the probability distribuion<br />
click = rbinom(k[1],1,prob=p)<br />
[/code]</p>
<p>This is mostly for aesthetic reasons and the only reason we are doing this is to create some variance in the data. This variance will help us show the usefulness of the application during demo to stakeholders (if real data is not ready by that time). Again, in most cases we do not need to go to such lengths.</p>
