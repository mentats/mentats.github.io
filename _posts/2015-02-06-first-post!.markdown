---
layout: post
title:  "First post!"
bit:    "Testing if this would work. This bit is a little subtitle for the post, I'm not liking the idea of showing just a part of the contents of the post."
date:   2015-02-06 20:13:07
categories: general
---
You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

To add new posts, simply add a file in the `_posts` directory that follows the convention `YYYY-MM-DD-name-of-post.ext` and includes the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets:

{% highlight SQL %}

DROP FUNCTION FN_EPOCH_TO_DURATION

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
   for the first three variables defined at the beginning of the funciton. The second SELECT
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
      WHEN (@format = 'W') AND ((@inputSeconds >= @oneWeek) AND ((@inputSeconds - ((@inputSeconds / (@oneWeek)) * @oneWeek)) / @oneDay) != 1) THEN CAST((@inputSeconds - ((@inputSeconds / (@oneWeek)) *   (@oneWeek))) / @oneDay AS VARCHAR(4)) + ' days'
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

SELECT DBO.FN_EPOCH_TO_DURATION('d',86399)

{% endhighlight %}

Check out the [Jekyll docs][jekyll] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll’s dedicated Help repository][jekyll-help].

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
