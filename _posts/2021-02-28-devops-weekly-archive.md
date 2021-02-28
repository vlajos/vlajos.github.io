---
modified: 2021-02-28
tags: [cloudflare, devops, github, jekyll, algolia]
title: "Setting up Devops Weekly newsletter's archive"
excerpt: "Some details about setting up a static website from scratch."
---

I am a subscriber of the [Devops weekly newsletter](https://www.devopsweekly.com/).
I was wondering a few weeks ago whether it had any web-based archive or not.
It would have been more convenient for me to read it on the web than in my mailer.
I asked its author whether it had any public archives, and also if he would mind if I created one.
He confirmed that it had not had and he gave its approval.

I am a subscriber since 2015, so I have a few hundred issues.
Unfortunately not all of them, but enough to start to build something meaningful.

My goal was to build something simple based on technologies that I knew or I was relatively confident that they were not going
to need lots of time to learn. I wanted to minimize the costs, but also wanted something stable.

I used previously [GitHub pages](https://pages.github.com/) with [Jekyll](https://jekyllrb.com/).
GitHub pages offers you free [git](https://git-scm.com/)-backed hosting.
So it comes implicitly with version tracking and backups.
Also, anybody can open Pull Requests to a public repository so potentially others could contribute easily later.
I thought the best is to keep the project open anyway so
I registered a new [GitHub organisation](https://github.com/devopsweeklyarchive/) and
I created a [repository](https://github.com/devopsweeklyarchive/devopsweeklyarchive.github.io) for the project.
GitHub pages automatically recognises if a repository is named as your `<account>.github.io` and it starts to serve its content
under the same domain. [devopsweeklyarchive.github.io](devopsweeklyarchive.github.io) in our case.

[Jekyll](https://jekyllrb.com/) is a ruby-based static site generator templating engine.
At least from this project's point of view, these are its most relevant features.
Also, it has some nice [free templates](https://jekyllthemes.io/free).
I used previously [Minimal Mistakes](https://github.com/mmistakes/minimal-mistakes) which is one of the most popular.
It has multiple options for how you can use it in a project, but [this starter template](https://github.com/mmistakes/mm-github-pages-starter/generate)
seemed to be the simplest. So I recreated the project repository from this template.

Configuring Minimal Mistakes is quite [straight-forward](https://mmistakes.github.io/minimal-mistakes/docs/configuration/).
Mainly it means filling its [_config.yml](https://github.com/devopsweeklyarchive/devopsweeklyarchive.github.io/blob/master/_config.yml).
Which is quite well documented.

{% highlight yaml %}
...
title: MM
email:
description: >- # this means to ignore newlines until "baseurl:"
  Write an awesome description for your new site here. You can edit this
  line in _config.yml. It will appear in your document head meta (for
  Google search results) and in your feed.xml site description.
twitter_username: username
github_username: username
minimal_mistakes_skin: default
search: true

# Build settings
markdown: kramdown
remote_theme: mmistakes/minimal-mistakes
# Outputting
permalink: /:categories/:title/
paginate: 5 # amount of posts to show
paginate_path: /page:num/
timezone: # https://en.wikipedia.org/wiki/List_of_tz_database_time_zones
...
{% endhighlight %}

Also, I had to delete the example posts from the `_posts` folder.
Essentially this is all you need to build an empty site.

To test it locally, you should run this command:

{% highlight bash %}
$ bundle exec jekyll serve --watch --incremental
{% endhighlight %}

Its output looks like this:

{% highlight bash %}
Configuration file: /home/vlajos/devopsweeklyarchive.github.io/_config.yml
            Source: /home/vlajos/devopsweeklyarchive.github.io
       Destination: /home/vlajos/devopsweeklyarchive.github.io/_site
 Incremental build: enabled
      Generating... 
      Remote Theme: Using theme mmistakes/minimal-mistakes
       Jekyll Feed: Generating feed for posts
                    done in 27.228 seconds.
/usr/lib/ruby/vendor_ruby/pathutil.rb:502: warning: Using the last argument as keyword parameters is deprecated
 Auto-regeneration: enabled for '/home/vlajos/devopsweeklyarchive.github.io'
    Server address: http://127.0.0.1:4000
  Server running... press ctrl-c to stop.
{% endhighlight %}

It serves the website on the mentioned url: `http://127.0.0.1:4000`.

My local ruby installation was not in the best shape, but reinstalling the related modules fixed its problems.

The next step was to convert the mails into [Markdown](https://en.wikipedia.org/wiki/Markdown)
files.
I gathered them into a single mailbox first and ended to use a set of scripts:

`split_big_mbox_to_individuals.sh`

As its name says, this one is just splitting the big mailbox to single files/issues with `formail`.
{% highlight bash %}
cat bigmbox | formail -ds sh -c \
    'cat > mails/msg.$FILENO'
rm $(grep -rl MAILER-DAEMON mails/)
{% endhighlight %}

`mbox_to_original.sh`

is stripping down the unneeded bits from the mails.
Most of the mail headers and the mailing list footers are quite pointless on the web.
Also, it names the files based on the issue number extracted from the `Subject` line.

{% highlight bash %}
source_file=mails/$1
id=$(grep -e ^Subject $source_file |\
    sed \
        -e 's/.*#//g' \
        -e 's/\?=//g' \
)
destination_file=originals/$id.mail.txt

grep -e ^Subject -e ^Date \
    <$source_file \
    >$destination_file

echo >>$destination_file

sed -e '1,/^$/ d' -e '/^--$/,$d' \
    <$source_file \
    >>$destination_file
{% endhighlight %}

`original_to_post.sh`

is responsible for the main Markdown conversion:
{% highlight bash %}

# Extracts the issue number from the file name.
id=$(echo $input|sed -e 's/.mail.txt//g' -e 's/.*\///g')

# These ones construct date variables from the Date header.
date=$(grep Date: $input|sed -e 's/Date: //g')
day=$(date --date "$date" '+%Y-%m-%d')
sec=$(date --date "$date" '+%Y-%m-%dT%H:%M:%S%:z')

# We generate the post name from the issue's date and number:
filename=../devopsweeklyarchive.github.io/_posts/$day-$id.md

# The title comes from the subject with escaping the # character.
# The mails were stored originally in quoted-printable encoding,
# so I needed to convert them to plain text.
title=$(\
    perl -MMIME::QuotedPrint -pe '$_=MIME::QuotedPrint::decode($_);' \
    <$input|\
    sed -e '0,/^$/d' -e '/^$/,$d' -e 's/#/\\#/g'|\
    tr '\n' ' '\
)

# Then we put together the post's header section:

echo '---' >$filename
echo "title: $title" >>$filename
echo "date: $sec" >>$filename
echo '---' >>$filename

# And its body: (I have injected some line-breaks and
# comments to make it a bit more readable.
# Those make it syntactically wrong.)

# Convert Quoted printable
perl -MMIME::QuotedPrint -pe '$_=MIME::QuotedPrint::decode($_);' \
    <$input |\

# Remove header section
    tail +5|\

# Remove standard footer
    grep -v 'If you received this email directly then' |\

# Remove double line-breaks before links
    perl -0 -p -e 's/\n\n *(http)/\n$1/g' |\

# Add explicit line-break before links
    perl -0 -p -e 's/\n(http)/\n<br>$1/g' |\

# Convert links to Markdown links
    sed -e 's/\(https\?:\/\/[^ ]*\)/[\1](\1)/g' \
    >>$filename

{% endhighlight %}

With these scripts, I could populate the `_posts` folder with the "main" content.
I also added a simple [about](https://devopsweeklyarchive.com/about/) page.

I uploaded this version since it had the main parts in place already.

Then I tweaked it a bit more.

I did not like the long URL [https://devopsweeklyarchive.github.io/](https://devopsweeklyarchive.github.io/),
so I have bought a bit shorter one: [https://devopsweeklyarchive.com/](https://devopsweeklyarchive.com/).
Fortunately, GitHub pages supports these alternative names, just a
[CNAME file](https://github.com/devopsweeklyarchive/devopsweeklyarchive.github.io/blob/master/CNAME) has to be provided.

GitHub pages offers now HTTPS for the domains with alternative names, but I also wanted to add
[Cloudflare](https://www.cloudflare.com/) to the picture.
You can say it is overkill, but still, this is a hobby project, I wanted to play a bit with that as well.
Also if you can have [DDOS protection](https://www.cloudflare.com/ddos/) and
world-wide [CDN](https://www.cloudflare.com/cdn) for free, then why not?
Anyway, it was still the same simplicity to configure it as a few years ago.
Originally I wanted to buy the domain from them as well, but currently, they offer transfer only.
Although [this page](https://www.cloudflare.com/products/registrar/) sounds quite attractive.

And a few more cherries to the top of the cake.

I wanted to add the basic Google management tools so registered for [Analytics](https://analytics.google.com/analytics/web/)
and [Search Console Tools](https://search.google.com/search-console/about).
Connecting them with the Minimal Mistakes template was very simple, just these lines in the
[configuration](https://github.com/devopsweeklyarchive/devopsweeklyarchive.github.io/blob/master/_config.yml#L121):

{% highlight yaml %}
...
google_site_verification: "Google site verification code"

analytics:
  provider: "google-gtag"
  google:
    tracking_id: "Analytics account ID"
    anonymize_ip: false # default
...
{% endhighlight %}

I wanted to create a site map as well since the site structure is very basic, but when I [checked](https://devopsweeklyarchive.com/sitemap.xml),
Minimal Mistakes already generated that for me.

The last piece was to make the site search a bit smarter.
Minimal Mistakes came with a
[default search engine](https://mmistakes.github.io/minimal-mistakes/docs/configuration/#lunr-default):
[Lunr](https://lunrjs.com/),
but it searched only in the first 50 words and it was not very sophisticated. 

So I choose to enable [Algolia](https://www.algolia.com/).
Fortunately, it is fully supported both with [Minimal Mistakes](https://mmistakes.github.io/minimal-mistakes/docs/configuration/#algolia)
and [Jekyll](https://github.com/algolia/jekyll-algolia).

And then I reached the project's most puzzling problem.
Why do these 300 pages generate 10000+ objects in Algolia?
I needed the help of Algolia's support and I had to dig a bit in its internals to understand
that all `<p>` tags are going to be indexed separately to be able to search them individually.
They describe this process [here](https://community.algolia.com/jekyll-algolia/options.html#nodes-to-index).
Also, the original newsletter was organised to have two line-breaks between the "body" of an article and the source link of the article.
Which meant that Jekyll generated two `<p>` tags from them.
Understanding this took a bit long, but then removing one of the line-breaks was quite simple.

Another tricky question is how to integrate the indexing step into a static system?
Luckily, they have a [nice solution](https://community.algolia.com/jekyll-algolia/github-pages.html#configuring-travis)
to this problem.
You can configure [Travis](https://travis-ci.com/) to execute some steps triggered by Git pushes.
Travis is a much smarter [CI](https://en.wikipedia.org/wiki/Continuous_integration)
system than this, but to solve this particular problem, we do not need its other capabilities.
Essentially I needed to copy their
[example Travis file](https://community.algolia.com/jekyll-algolia/github-pages.html#configuring-travis)
to the
[new repository](https://github.com/devopsweeklyarchive/devopsweeklyarchive.github.io/blob/master/.travis.yml).
So now we have smart typo-resistant search functionality.

Finally, the [Devops weekly newsletter](https://www.devopsweekly.com/) got its
[public archive](https://devopsweeklyarchive.com/).
