---
modified: 2015-07-02
tags: [protractor, Browserstack, node.js, jenkins]
title: "Jenkins, Browserstack integration: Link jenkins builds with Browserstack sessions"
excerpt: "Linking Browserstack session's in Jenkins' build page using the Description setter plugin and a bit javascript"
---
We have a couple of Jenkins tasks for running tests with Browserstack.
One suite contains 5-6 tests and 4 platforms configured with Jenkins' matrix plugin.
(We plan to increase both dimensions.)

If any of the tests fail and we want to investigate what happened we have to open a Browserstack window, navigate to the
given build page and find on the page the given session.
This can be painful if the suite is still running because Browserstack is continuously refreshing the page.
When the tests are done at least the page is stable, so you can search with the browser's find function.
If the test had not run amongst the first ten of the given suite you have to click one more.

This is not so bad if you do it once or twice a day. Approximately 30 seconds and 4-5 clicks.
But if you develop intensively and the tests are not stable yet this could take 10% of your time maybe even more.

Therefore we wanted to automate somehow this process. It did not seem to be too complex,
Jenkins already communicates with Browserstack, though it does indirectly, but maybe we could send back the
session's link to it.

We asked Browserstack's support. They told us that we could access the remote session's id and
with that id, we could also get the session's url using Browserstack's REST API.

Getting the webdriver session's data:
{% highlight javascript %}
    browser.driver.session_.then(function(sessionData) {
        console.log(sessionData);       // All session data
        console.log(sessionData.id_);   // The webdriver session id
    });
{% endhighlight %}

And we can call the REST API using this id on the end-point:
{% highlight javascript %}
  var link= 'https://www.browserstack.com/automate/sessions/' + sessionData.id_ + '.json';
{% endhighlight %}

Unfortunately, this second link is protected with HTTP-Auth.
But the expected login/password here is not the API credentials, but the web user's one.
So we have to allow Jenkins to use somehow one of our web user's credentials.

The final code for this:
{% highlight javascript %}
browser.driver.session_.then(function(sessionData) {
  require('https').request({
    host: 'www.browserstack.com',
    port: 443,
    path: '/automate/sessions/' + sessionData.id_ + '.json',
    method: 'GET',
    auth: 'login:password'
  },function(res){
    res.setEncoding('utf8');
    res.on('data', function(d) {
      var sessionUrl = JSON.parse(d).automation_session.browser_url;
      console.log('Browserstack session url: ' + sessionUrl);
    });
  }).end();
});
{% endhighlight %}

In our case, we displayed the capabilities as well like in the first snippet and we have an exception when we test locally
instead of Browserstack and when the password is accessed somehow differently.
So it is a bit more complex but this is the main concept.

You can place this snippet anywhere in your tests. If you want to run it for every test probably the best place is a centralised code.
We added this to our `protractor.conf`'s `onprepare` section.

After implementing this we have got a line in our logs like this:
`Browserstack session url: https://www.browserstack.com/automate/builds/123.../sessions/456...`

When we started to develop this function we did not know yet how we could use this link in Jenkins, so originally this was intended to be only a temporary verification step.
Fortunately Jenkins has a [Description Setter Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Description+Setter+Plugin){:target="_blank"}
which can run regular expression against the build log and it can use their results customise the given build's description.

You have to install the plugin first with these steps:

 * Manage Jenkins
 * Manage Plugins
 * Available
 * Description setter plugin

Then you have to configure it for all the tasks where you want to use it:

 1. Open the task's configuration page.
 2. Scroll down to the `Add post-build action`'s and add the plugin's call.
 3. We use with the following settings:
 *  Regular expression: `Browserstack session url: (.*)`
 *  Description: `<a href="\1">Browserstack session</a>`
 *  Regular expression for failed builds: `Browserstack session url: (.*)`
 *  Description for failed builds: `<a href="\1">Browserstack session</a>`

The last two items appear when you open the advanced options. Without them, this feature is pretty useless for debugging...
We tried to add `target="_blank"`, but it was stripped out by Jenkins. Probably it could be added somehow...

And the result looks like this in the history:

> #661 Jul 02, 2015 2:09 PM
> Browserstack session

