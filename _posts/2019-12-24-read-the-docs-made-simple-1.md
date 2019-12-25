---
title:  "Read The Docs for REST APIs Made Simple, Part 1"
date:   2019-12-24 12:40:37
classes: wide
author_profile: false
toc: true
toc_sticky: true
toc_label: <a href="#site-nav"><font color="#eadcb5">On This Page</font></a>
header:
  image: /assets/images/Sphinx.png
  teaser: /assets/images/Sphinx.png
categories:
  - Programming
tags: 
  - Read The Docs
  - Sphinx
  - Documentation
---

While developing software, documentation is frequently the furthest thing from a programmer's mind. Writing documentation can be tedious and, well let's just say it, downright boring.  But these days, open platforms supporting public APIs are proliferating and their success depends heavily on how easy it is to use these APIs.  There's no getting around it, if you have created an API for your service that you want your customers to adopt, you have to have good documentation for it.

I've written a lot of documentation in my day, and continue to do so, using various tools like emacs, Word, PowerPoint, Confluence, etc.  My favorite though is [Read The Docs](https://docs.readthedocs.io/en/stable/){:target="_blank"}.  With it you can create visually attractive and well organized documents with (relatively) simple textual directives that can produce HTML and PDF copy for online presentation. 

This blog is part 1 of a 3 part series which illustrate how to create a simple **Read The Docs** online document ib HTML and PDF formats. In part 1, I'll cover making a basic document from scratch that will form the basis for the project you'll flesh out in parts 2 and 3. 

## Install Sphinx

The key to harnessing **Read The Docs** is getting your head wrapped around the [reStructured Text](https://docutils.sourceforge.io/rst.html){:target="_blank"} (RST) format.  RST is a form of text markdown that provides directives to help you organize and format your document using a simple text editor.  To create a document with a set of RST files, you use the [Sphinx](https://www.sphinx-doc.org/en/1.5/tutorial.html){:target="_blank"} compiler. 

Let's get started with Sphinx by creating a starter project that will be built out into a full-fledged REST API document during the course this series of articles. Sphinx is Python based tool, so you will have to have Python running on your system. Create the project directory then install Sphinx in a Python virtual environment:

{% highlight bash linenos%}
% mkdir starterdoc
% cd starterdoc
% pip install pipenv
% pipenv shell
(starterdoc) % pip install sphinx
{% endhighlight %}


## Create Your First Document

Next create the starter project base documents, run the `sphinx-builder` command, answering the questions that follow it.  You can use your own name when prompted for the document author in line 16.

{% highlight bash linenos%}
(starterdoc) % sphinx-builder restdoc
Welcome to the Sphinx 2.3.1 quickstart utility.

Please enter values for the following settings (just press Enter to
accept a default value, if one is given in brackets).

Selected root path: restdoc

You have two options for placing the build directory for Sphinx output.
Either, you use a directory "_build" within the root path, or you separate
"source" and "build" directories within the root path.
> Separate source and build directories (y/n) [n]: y

The project name will occur in several places in the built documentation.
> Project name: Rest API
> Author name(s): Vic Hargrave
> Project release []: 0.1

If the documents are to be written in a language other than English,
you can select a language here by its language code. Sphinx will then
translate text that it generates into that language.

For a list of supported codes, see
https://www.sphinx-doc.org/en/master/usage/configuration.html#confval-language.
> Project language [en]:

Creating file restdoc/source/conf.py.
Creating file restdoc/source/index.rst.
Creating file restdoc/Makefile.
Creating file restdoc/make.bat.

Finished: An initial directory structure has been created.

You should now populate your master file restdoc/source/index.rst and create other documentation
source files. Use the Makefile to build the docs, like so:
   make builder
where "builder" is one of the supported builders, e.g. html, latex or linkcheck.
{% endhighlight %}

The directory structure of the `starterproject` will look like this:

{% highlight bash%}
.
├── Pipfile
├── Pipfile.lock
└── restdoc
    ├── Makefile
    ├── build
    ├── make.bat
    └── source
        ├── _static
        ├── _templates
        ├── conf.py
        └── index.rst
{% endhighlight %}

File you edit to create your document are contained in the `source` directory. To build your first document:

{% highlight bash%}
(starterdoc) % cd restdoc
(starterdoc) % make clean html
{% endhighlight %}

The directory structure of you project will then look like this:

{% highlight bash%}
.
├── Makefile
├── build
│   ├── doctrees
│   │   ├── environment.pickle
│   │   └── index.doctree
│   └── html
│       ├── _sources
│       │   └── index.rst.txt
│       ├── _static
│       │   ├── alabaster.css
│       │   ├── basic.css
│       │   ├── custom.css
│       │   ├── doctools.js
│       │   ├── documentation_options.js
│       │   ├── file.png
│       │   ├── jquery-3.4.1.js
│       │   ├── jquery.js
│       │   ├── language_data.js
│       │   ├── minus.png
│       │   ├── plus.png
│       │   ├── pygments.css
│       │   ├── searchtools.js
│       │   ├── underscore-1.3.1.js
│       │   └── underscore.js
│       ├── genindex.html
│       ├── index.html
│       ├── objects.inv
│       ├── search.html
│       └── searchindex.js
├── make.bat
└── source
    ├── _static
    ├── _templates
    ├── conf.py
    └── index.rst
{% endhighlight %}

As you can see, the `build` folder contains the target files in the `html` directory. All the scaffolding for the HTML document are added automatically, including a search capability that enables your readers to search your documentation. The `html/index.html` is the root page and is build from `source/index.rst`. To view the document just open it in a browser:

{% highlight bash%}
(starterdoc) % open build/html/index.html
{% endhighlight %}

Your first document will look something like this:

![First Sphinx Document](/assets/images/First_Sphinx_Document.png){:width="100%" .align-center}

## Next Steps

In this blog you were introduced to the process of creating a basic Sphinx project and how to generate your first document.  Part 2 of this series will delve more deeply into creating a table contents and adding pages to your document to express nicely formatted REST APIs.
