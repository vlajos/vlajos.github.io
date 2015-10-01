---
layout: post
modified: 2015-07-21
comments: true
tags: [angular.js, javascript, $timeout, $interval, protractor, testing]
title: "Testing an alert was more complex than it seemed to be..."
excerpt: "Alternative for $timeout to make protractor testing a bit easier."
---
Currently we are working on a big refactoring project. We are rewriting a big chunk of legacy javascript with Angular.js.
We also write end to end tests with protractor to keep our system more robust.

My colleagues used UI Bootstrap module's [alert](https://angular-ui.github.io/bootstrap/#/alert)
function to display different things to our clients.
And they wanted to test if the alert is displayed properly.

The first attempts returned with a strange error message:
{% highlight javascript %}
10:55:37.099 WARN - Exception thrown org.openqa.selenium.TimeoutException: asynchronous script timeout: result was not received in 11 seconds
{% endhighlight %}

Basically the test clicked a button and wanted to check if there is an alert or not.
It was very strange that - based on the logs - the click happened properly, but if we removed anything after the `click()` action the timeout remained still there.
It took a bit of time to understand what happened in the background.

Unfortunately the alert's internal dismiss functionality has been implemented with Angular's `$timeout` service which increments
`$browser`'s outstanding request count and protractor with `ignoreSynchronization=false` waits correctly until there is any outstanding requests.
This means if you do not disable the synchronization, you cannot see the opened alert, because protractor suspends execution until it is visible.

So in our case the `click()` opened the alert, which started the `$timeout`, and protractor suspended the execution and then it timed out.

This [link](https://github.com/angular/protractor/issues/169)
helped us a lot to understand the situation:

Therefore if you want to test any `$timeout` based functionality you have to turn on `ignoreSynchronization` which is basically a hack for non-Angular apps.

[There](https://github.com/angular/angular.js/commit/2b5ce84fca7b41fca24707e163ec6af84bc12e83)
they suggest to use `$interval` because it fits better to this case.

As a work around currently we turned on `ignoreSynchronization`, but for a proper solution we implemented the suggested `$interval` transition
and sent a [Pull Request](https://github.com/angular-ui/bootstrap/pull/3982).

It seems that there are different opinions regarding this `$timeout` vs `$interval` debate. (See the comments in the PR.)
