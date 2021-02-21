---
modified: 2015-06-16
tags: [pdf, ghostscript, fpdi, concatenate, orientation, rotate]
title: "Save our day with ghostscript to concatenate PDF files"
excerpt: "Using ghostscript to manipulate PDF files."
---
This issue started without any extra notice.
Customer service notified us that labels were not uploaded.
Essentially, we receive the labels from our logistic partners, we print them out and apply them to the parcels we send out.
And there is also a small step in the system where we concatenate the label batches to bigger files to make
the printing easier for our colleagues.
Both the input and the output files are simple PDFs.

When we checked our logs we noticed that something had been changed with the original labels.
{% highlight php %}
[... 11:01:06 2015] [error] [client 333.333.333.333] PHP Fatal error:
Uncaught exception 'Exception' with message 'This document (/tmp/label_55a780e22172f.pdf) probably uses a compression technique which is not supported by the free parser shipped with FPDI. (See https://www.setasign.com/fpdi-pdf-parser for more details)' in
...lib/FPDI-1.5.2/pdf_parser.php:329
Stack trace:
#0 ...lib/FPDI-1.5.2/pdf_parser.php(202): pdf_parser->_readXref(Array, 87345)
#1 ...lib/FPDI-1.5.2/fpdi_pdf_parser.php(71): pdf_parser->__construct('/tmp/...')
#2 ...lib/FPDI-1.5.2/fpdi.php(128): fpdi_pdf_parser->__construct('/tmp/..')
#3 ...lib/FPDI-1.5.2/fpdi.php(108): FPDI->_getPdfParser('/tmp/...')
#4 ...php/utils/concatpdf.class.php(37): FPDI->setSourceFile('/tmp/...')
#5 ...php/utils/shipping/provider in ...lib/FPDI-1.5.2/pdf_parser.php on line 329
{% endhighlight %}

The error message was quite clear: the input PDF files' format had been changed.
We could verify it easily:

{% highlight bash %}
$ file label_55a780e22172f.pdf
label_55a780e22172f.pdf: PDF document, version 1.7
{% endhighlight %}

We checked the link the error message mentioned...
They have a commercial add-on to open PDF files higher than 1.4.

Fortunately, I played with ghostscript a few days ago and remembered that it could manipulate PDF files efficiently.
And Google gave us useful examples on stackoverflow.

We implemented the new code in less than half an hour. The main logic was very simple:

{% highlight bash %}
gs -dBATCH -dNOPAUSE -q -sDEVICE=pdfwrite -sOutputFile=$outfile $inputfiles
{% endhighlight %}

We gave it a try and it worked.
We tried it with longer files and we noticed that a few labels had changed their orientation.
2-3 of 60 labels were landscape instead of portrait, though all the originals were portrait.
And the problem appeared consistent with the same files.

We started to play with the switches... Ghostscript had much more options than we expected and its documentation is not really
Google-friendly, but we found finally the `AutoRotatePages` flag.

Our first attempt was the `-dAutoRotatePages=/All` setting, but this uses some kind of majority decision for rotating the pages...
We tried this with only one label which had been rotated unintentionally.
It did not work as we could expect.
It seemed that something like this had been enabled on page level.
At least with this given file ghostscript did the same rotation with or without the flag.

Fortunately, our next attempt was successful.
We added `-dAutoRotatePages=/None`
Which worked properly for both the big batches and the single files.

{% highlight bash %}
gs -dAutoRotatePages=/None -dBATCH -dNOPAUSE -q -sDEVICE=pdfwrite -sOutputFile=$outfile $inputfiles
{% endhighlight %}

A 27 years old software saved our day.

[Link to its version 1.n History](http://ghostscript.com/doc/current/History1.htm#Version1.0){:target="_blank"}
