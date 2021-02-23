# How html javascript excuting
## Downloading html and script for executing locally in chrome v8 engine.
**Let’s see what happens when we open the HTML file in the browser:**

```
Ref: https://www.telerik.com/blogs/journey-of-javascript-downloading-scripts-to-execution-part-i

The browser starts parsing the HTML code. When it comes across a script tag in the head section, 
the HTML parsing is paused. An HTTP request is sent to the server to fetch the script. 
The browser waits until the entire script is downloaded. It then does the work of parsing, 
interpreting and executing the downloaded script (we’ll get into the details of the entire process later in the article). 
This happens for each of the four scripts.

Once this is done, the browser resumes its work of parsing HTML and creating DOM nodes. 
```
