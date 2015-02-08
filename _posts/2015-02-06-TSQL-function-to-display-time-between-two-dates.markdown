---
layout: post
title:  "Simple TSQL function to display time difference between two dates"
bit:    "This simple function will help when trying to display the time difference between two dates. It works well to display SLM data too."
date:   2015-02-08 20:15:00
categories: general
---

Remedy stores dates in UNIX time / epoch / POSIX or whatever you want to call it. Dates are represented as `number of seconds since 1/1/1970`. This provides a number of advantages when making calculations with dates, but is a problem when trying to display dates to users un a human-readable way.

This post is not about how to convert epoch to date strings, there are already various sites that provide online converters and examples of SQL queries to do so, like [epochconverter.com][epc] or [esqsoft's date-to-epoch][esq].

This is about calculating the difference between two dates and formatting the output so anyone can read it. This is useful too if you want to display SLM data stored in seconds, as UpElapsedTime or DownElapsedTime from SLM_Measurement.

Let's take the next query as an example:

{% highlight SQL %}

SELECT h.Last_Resolved_Date, h.Submit_Date, h.Last_Resolved_Date - h.Submit_Date
FROM HPD_Help_Desk h
WHERE h.Incident_Number = 'INC000000000001'

{% endhighlight %}

Let's suppose the result of this query is `1310397002 | 1310219787 | 177215`. The result of subtracting the `Submit_Date` (1310397002) to the `Last_Resolved_Date` (1310219787) is 177215, meaning 177215 seconds passed between the creation and the resolution of that entry. The function this post is about will take this integer as an input parameter, and return a formatted string that a person can read. In this case, it would be used like this:

{% highlight SQL %}

SELECT DBO.FN_EPOCH_TO_DURATION('d', h.Last_Resolved_Date - h.Submit_Date)
FROM HPD_Help_Desk h
WHERE h.Incident_Number = 'INC000000000001'

{% endhighlight %}

The result of this query is `2 days, 01:13:35`.

Noticed the first input parameter for the function? For bigger inputs, I created this funciton with another parameter just in case you prefer to split the `days` part and display `weeks, days` instead. The first input parameter must be `d` or `w`. From now on, examples will be simplified by directly providing a number of seconds to the function instead of subtracting two dates:

{% highlight SQL %}

SELECT DBO.FN_EPOCH_TO_DURATION('d', 846234) -- This query returns '9 days, 19:03:54'
SELECT DBO.FN_EPOCH_TO_DURATION('w', 846234) -- This query returns '1 week, 2 days, 19:03:54'

{% endhighlight %}

You can create this function in your SQL Server DB with the following code:

{% highlight SQL %}

-- Create function FN_EPOCH_TO_DURATION (@inputs and TYPES) RETURNS output(LEN) AS
CREATE FUNCTION FN_EPOCH_TO_DURATION (@format CHAR(1), @inputSeconds INT) RETURNS VARCHAR(28) AS
BEGIN
   -- Variables are variable-length (VARCHAR instead of CHAR) so the output isn't filled with 
   -- whitespace and the resulting string is correctly formatted.
   DECLARE @weeks VARCHAR(10)
   DECLARE @days VARCHAR(10)
   DECLARE @hms VARCHAR(8)
   DECLARE @formattedString VARCHAR(28)

   -- For readability, variables containing the number of seconds in one week and one day
   -- are declared and will always be used for calculations instead of numbers if possible.
   DECLARE @oneWeek INT = 604800
   DECLARE @oneDay INT = 86400

   /* Each of the three following SELECT statements contains a case that sets the value
   for the first three variables defined at the beginning of the funciton. The last SELECT
   statement provides formatting based on @format input parameter, as follows:
   - "W" input will return "WW Weeks, DD days, HH:MM:SS"
   - "D" input will return "DD days, HH:MM:SS" 
   
   SQL CASE expressions break after a true condition, so line order is important in some cases
   e.g. the first SELECT statement has a CASE with three possible options. If the first one of 
   them is moved to the bottom, one of the others will be true first and passing a 'D' as the 
   first parameter when calling the function wil not work properly*/

   -- Calculate weeks
   SELECT @weeks = CASE
      -- If @format is D or if input is less than one week, return null.
      WHEN @format = 'D' OR (@inputSeconds < @oneWeek) THEN NULL
      -- If input is two or more weeks, return number of weeks + plural string:
      WHEN (@inputSeconds >= @oneWeek * 2) THEN CAST((@inputSeconds / @oneWeek) AS VARCHAR(4)) + ' weeks'
      -- If input is one week or more but less than two weeks, return singular string:
      WHEN (@inputSeconds >= @oneWeek AND @inputSeconds < (@oneWeek * 2)) THEN '1 week'
      -- Else return error:
      ELSE 'Error'
   END
   
   -- Calculate days, based on @format input parameter
   SELECT @days = CASE
      -- If @format is W:
      -- If input is less than one day OR if resulting modulus of (input / one week) is less than one day OR if input is multiple of one week, return null:
      WHEN (@format = 'W') AND ((@inputSeconds < @oneDay) OR (@inputSeconds % (@oneWeek) < @oneDay) OR (@inputSeconds % (@oneWeek) = 0)) THEN NULL
      -- If input is more than one week and resulting value of (input - (number of weeks * 7)) != 1, return number of days - (number of weeks * 7) + plural string:
      -- This one makes it so if the input is 8 days, the final output of the function is "1 week, 1 day..." instead of "1 week, 8 days..."
      WHEN (@format = 'W') AND ((@inputSeconds >= @oneWeek) AND ((@inputSeconds - ((@inputSeconds / (@oneWeek)) * @oneWeek)) / @oneDay) != 1) THEN CAST((@inputSeconds - ((@inputSeconds / (@oneWeek)) *  (@oneWeek))) / @oneDay AS VARCHAR(4)) + ' days'
      -- If input is two or more days but less than a week, return number of days + plural string:
      WHEN (@format = 'W') AND (@inputSeconds >= @oneDay * 2 AND @inputSeconds < @oneWeek) THEN CAST(@inputSeconds / @oneDay AS VARCHAR(4)) + ' days'
      -- If input is one day or more but less than two days OR if (input - (total weeks * 7)) = 1, return singular string:
      WHEN (@format = 'W') AND ((@inputSeconds >= @oneDay AND @inputSeconds < @oneDay * 2) OR (((@inputSeconds - ((@inputSeconds / (@oneWeek)) * (@oneWeek))) / @oneDay) = 1)) THEN '1 day'
      -- If @format is D:
      -- If input is two or more days, return number of days + plural string:
      WHEN (@format = 'D') AND (@inputSeconds >= @oneDay * 2) THEN CAST((@inputSeconds / @oneDay) AS VARCHAR(4)) + ' days'
      -- If input is one day or more but less than two days, return singular string:
      WHEN (@format = 'D') AND (@inputSeconds >= @oneDay) AND (@inputSeconds < @oneDay * 2) THEN '1 day'
      -- If input less than one day, return null:
      WHEN (@format = 'D') AND (@inputSeconds < @oneDay) THEN NULL
      -- Else return error:
      ELSE 'Error'
   END
   
   -- Calculate HH:MM:SS
   SELECT @hms = CASE
      -- If input is not multiple of one day, return formatted string:
      WHEN (@inputSeconds % @oneDay != 0) THEN CONVERT(VARCHAR(8), DATEADD(SECOND, @inputSeconds, 0), 108)
      -- If input is exactly one day OR if input is multiple of one day, return null:
      WHEN (@inputSeconds = @oneDay) OR (@inputSeconds % @oneDay = 0) THEN NULL
      -- Else return error:
      ELSE 'Error'
   END

   -- This last SELECT statement formats the output based on previous calculations
   SELECT @formattedString = CASE
      -- Errors shoultn't have occured, but are captured just in case.
      -- If there was an error calculating weeks, return error 1:
      WHEN @weeks = 'Error' THEN 'Error 1: bad weeks calc.'
      -- If there was an error calculating days, return error 2:
      WHEN @days = 'Error' THEN 'Error 2: bad days calc.'
      -- If there was an error calculating HH:MM:SS, return error 3:
      WHEN @hms = 'Error' THEN 'Error 3: bad HH:MM:SS calc.'
      -- If input is negative, return error 4:
      WHEN (@inputSeconds < 0) THEN 'Error 4: negative value'
      -- If input is 0, return 'Nothing':
      WHEN (@inputSeconds = 0) THEN 'Nothing'
      -- With weeks, days and time:
      WHEN (@weeks IS NOT NULL AND @days IS NOT NULL AND @hms IS NOT NULL) THEN @weeks + ', ' + @days + ', ' + @hms
      -- With weeks and days, no time:
      WHEN (@weeks IS NOT NULL AND @days IS NOT NULL AND @hms IS NULL) THEN @weeks + ', ' + @days
      -- With weeks and time, no days:
      WHEN (@weeks IS NOT NULL AND @days IS NULL AND @hms IS NOT NULL) THEN @weeks + ', ' + @hms
      -- With weeks, no time and no days:
      WHEN (@weeks IS NOT NULL AND @days IS NULL AND @hms IS NULL) THEN @weeks
      -- Without weeks, with days and time:
      WHEN (@weeks IS NULL AND @days IS NOT NULL AND @hms IS NOT NULL) THEN @days + ', ' + @hms
      -- Without weeks, with days and no time:
      WHEN (@weeks IS NULL AND @days IS NOT NULL AND @hms IS NULL) THEN @days
      -- Without weeks, with time and no days:
      WHEN (@weeks IS NULL AND @days IS NULL AND @hms IS NOT NULL) THEN @hms
      -- Else soluld not occur either, will be considered error 5 for now:
      ELSE 'Error 5: check function def.'
   END
   -- Return formatted string
   RETURN @formattedString
END

{% endhighlight %}

If the input is less than `604800` (seconds in one week) the resulting formatted string will be the exactly the same regardless of wether `d` or `w` was provided as parameter.

Some sample uses:

{% highlight SQL %}

SELECT DBO.FN_EPOCH_TO_DURATION('d', 362) -- This query returns '00:06:02'
SELECT DBO.FN_EPOCH_TO_DURATION('w', 362) -- This query returns '00:06:02'
SELECT DBO.FN_EPOCH_TO_DURATION('d', 86400) -- This query returns '1 day'
SELECT DBO.FN_EPOCH_TO_DURATION('w', 86400) -- This query returns '1 day'
SELECT DBO.FN_EPOCH_TO_DURATION('d', 368322) -- This query returns '4 days, 06:18:42'
SELECT DBO.FN_EPOCH_TO_DURATION('w', 368322) -- This query returns '4 days, 06:18:42'
SELECT DBO.FN_EPOCH_TO_DURATION('d', 604800) -- This query returns '7 days'
SELECT DBO.FN_EPOCH_TO_DURATION('w', 604800) -- This query returns '1 week'
SELECT DBO.FN_EPOCH_TO_DURATION('d', 1427843) -- This query returns '16 days, 12:37:23'
SELECT DBO.FN_EPOCH_TO_DURATION('w', 1427843) -- This query returns '2 weeks, 2 days, 12:37:23'

{% endhighlight %}

You can get this function without the comments or translated to spanish at the [Remedy-ITSM-SQL-tools][rep] repository. I'd love if this repo would get some people's contribution, it'd be great if anyone would translate this to another language and open a pull request, I'll review and accept it as soon as I can. 

This function was written to work with MS SQL Server 2008, but it's very simple and I guess it would work in SQL Server 2005 without a problem. Modifying it to work with an Oracle DB should be very simple too, but I don't have an Oracle DB for testing at the moment. If anyone gets this to work with an Oracle DB please share and I'll update the repo.

Not much more to say about this. I wish you all a good week!

[epc]: http://www.epochconverter.com
[esq]: http://www.esqsoft.com/javascript_examples/date-to-epoch.htm
[rep]: https://github.com/mentats/Remedy-ITSM-SQL-tools
[fdl]: http://sqlfiddle.com/#!4/8e249/2