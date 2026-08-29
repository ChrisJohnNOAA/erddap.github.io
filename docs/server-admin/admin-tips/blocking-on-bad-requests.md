# Blocking ERDDAP™ requests based on the content of the request, not based on the IP

This content is based on a [message from Roy Mendelssohn to the ERDDAP™ users group](https://groups.google.com/g/erddap/c/XcvPkoGtchg).

## The problem 

If you are like us,  you are seeing a lot of bots making data requests to your ERDDAP™,  and the requests often seem like they are done by poorly coded scripts,  whether by LLMs (my guess) or by humans.  One thing about LLMs is they will do exactly what you prompt them to do,  so if you don’t tell them to have the script check return codes, and don’t  tell them what to do in different cases,  then usually the code will not do either of those things.  And if you tell it to keep on trying,  it will.  Which is what we are seeing at least.

But the ones that are doing a number on our ERDDAP™ are like this:

https://coastwatch.pfeg.noaa.gov/erddap/jplMURSST41.parquetWMeta

These send ERDDAP™ into never-never land, due,  to the best that I can determine,  that this URL is not only requesting the entire dataset,  which is I don’t know how many terabytes,  but wants it converted to a parquet file.  Worse the requests are coming in from moving IPs,  so blocking IPs is worse than whack-a-mole,  you will never get them all blocked.

So the question is how can you block based on the content of the request?  Before going further,  if you are experience crashes it may be from different causes then what we are seeing - I spent a lot of time going through logs and also I get notified when there is dangerous high memory use,  and noted when a crash followed that notification,  and also noticed what was the common denominator in these crashes,  which may not be the case for your server.  So research this before taking any action,  but this may  give you some idea how to block these requests. 

Just so I am clear,  you can think of an ERDDAP™ request as:

https://baseURL/erddap/method/datasetID.filetype?constraint

In our case the common denominator of the crashes were griddap or tabletop requests with certain filetypes but no constraint.  So the question is how to block requests without a constraint?  Except it is not  that simple,  because there are any number of filetypes that don’t need a constraint and will be well-behaved, so we don’t want to block those.  After talking to Chris,  and no doubt I have left out something,  the filetypes that are well behaved without a constraint are:

- .croissant
- .iso19115_2
- .iso19139_2007
- .iso19115_3_2016
- .ncCFHeader
- .ncCFMAHeader
- .das
- .dds
- .html
- .graph
- .subset
- .ncHeader
- .help
- .fgdc
- .iso19115
- .ncoJsonHeader (this one will be in the upcoming new release).

## Solution

The solution "I found" works in Apache2,  I do not know nginx but I imagine there is something similar.  At first I tried mod_security,  but that got too complicated and didn’t work very well.  The solution is to use mod_rewrite.  Now I am anything but an expert on this, so while I defined what I was trying to accomplish,  the solution, as well as the explanation,  is due to Claude.ai.  ChatGPT will supply you with basically the same answer.

Step 1.  Make certain that mod_rewrite is installed and enabled.  Since how to do this varies with OS,  this is something you can ask your favorite chatbot.

Step 2.  In the appropriate file that configures ssl for apache2 (which again varies by OS), for example it might be something like /etc/apache2/sites-enabled/ssl.conf,  add the following under the appropriate VirtualHost definition (note if you copy this there are only 4 lines,  the third line may be wrapped,  unwrap it)

```
RewriteEngine On
RewriteCond %{QUERY_STRING} ^$
RewriteCond %{REQUEST_URI} ^/erddap/(griddap|tabledap)/[^/?]+\.(?!(?:croissant|iso19115_2|iso19139_2007|iso19115_3_2016|ncCFHeader|ncCFMAHeader|das|dds|html|graph|subset|ncHeader|help|fgdc|iso19115|ncoJsonHeader)$)[A-Za-z0-9_]+$
RewriteRule ^ - [R=429,L]
```

Step 3.  Check that the configuration is valid:  `sudo apache2ctl configtest`

Step 4.  Restart apache2:  `sudo systemctl restart apache2`

Step 5.  Check your logs that nothing is being blocked that should not be,  and that appropriate requests without a constraint return a 429 without ever hitting your tomcat

## Explanation

Why does this work and what does this do - here is the explanation from Claude.ai:

Line 1 — `RewriteEngine On`
Turns on mod_rewrite processing for this scope. Without it, the RewriteCond/RewriteRule directives below are simply ignored.

Line 2 — `RewriteCond %{QUERY_STRING} ^$`
A condition that must be true before the rule below applies. `%{QUERY_STRING}` is everything after the ? in the request URL. ^$ is a regex meaning "start of string immediately followed by end of string" — i.e., an empty string. So this condition is true only when there's no query string at all — no subsetting/constraint expression on the request.

Line 3 — `RewriteCond %{REQUEST_URI} ^/erddap/(griddap|tabledap)/[^/?]+\.(?!(?:...)$)[A-Za-z0-9_]+$`
A second condition, checked against `%{REQUEST_URI}` — the literal request path as the client sent it, always the full path regardless of where in the config this rule lives (deliberately chosen over letting the RewriteRule pattern itself do the matching, because pattern-matching inside a `<Location>` block can behave ambiguously — see note below). Breaking down the regex:

`^/erddap/` — must start with /erddap/
(griddap|tabledap)/ — followed by one of the two ERDDAP™ access methods

`[^/?]+` — the datasetID: one or more characters that aren't / or ?

`\.` — a literal dot

`(?!(?:croissant|iso19115_2|...|ncoJsonHeader)$)` — a negative lookahead: "as long as what follows is not one of these exact fileType names all the way to the end of the string." These are the fileTypes ERDDAP™ can legitimately serve with no constraint (metadata, structure, form pages, etc.) — the lookahead is what excludes them from being blocked.

`[A-Za-z0-9_]+$` — the actual fileType extension (letters, digits, underscore), required to run to the end of the string.
So this condition is true only when the path is a griddap/tabledap request for some fileType that isn't on the safe-without-constraint list.

Line 4 — `RewriteRule ^ - [R=429,L]`
The rule itself. Because both conditions above must already be true for Apache to even evaluate this line, the pattern here doesn't need to check anything else — ^ just matches "start of the string," which is always true. - means "don't rewrite the URL to anything different" (we're not redirecting anywhere, just short-circuiting the request). The flags:

`R=429` — respond with an HTTP redirect-class action carrying status code 429 ("Too Many Requests") instead of serving the request.
L — "Last": stop processing any further rewrite rules once this one fires.
Put together: if the query string is empty, AND the request is for a griddap/tabledap fileType that isn't on the safe-unconstrained list, immediately return 429 — without ever contacting the ERDDAP/Tomcat backend.

Why `%{REQUEST_URI}` instead of letting the RewriteRule pattern match the path directly (worth including as a note for colleagues, since it's the non-obvious part): inside a `<Location>` block, what a bare RewriteRule pattern actually gets matched against can behave inconsistently depending on Apache version and context. Explicitly pulling the full path via RewriteCond `%{REQUEST_URI}` sidesteps that ambiguity entirely — it's always the literal, complete request path, so the regex behaves exactly as written regardless of where the rule is nested.


I have been using this for several days now and it appears to work very well,  it is blocking what I am trying to block and not blocking what I don’t want to block.  And our ERDDAP™ has become much more stable.