As Ron Garrity says, you can use Curl like this:

curl -Iv http://www.aptivate.org 2>&1 | grep -i 'connection #0'
And it outputs these two lines if keep-alive is working:

* Connection #0 to host www.aptivate.org left intact
* Closing connection #0
And if keep-alive is not working, then it just outputs this line:

* Closing connection #0
The output Connection ... left intact proves that the server did not close the connection, and it is available for the client to reuse. It's up to the client to decide whether it actually wants to reuse the connection or not. You can demonstrate it with Curl by listing the same URL twice on the command line

curl -Iv http://www.aptivate.org --next http://www.aptivate.org 2>&1 | grep -i '#0'
in which case it will give output something like:

Re-using existing connection! (#0) with host ...