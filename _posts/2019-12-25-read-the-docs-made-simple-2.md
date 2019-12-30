---
title:  "Read The Docs for REST APIs Made Simple, Part 2"
date:   2019-12-25 12:40:37
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

[Read The Docs](https://docs.readthedocs.io/en/stable/){:target="_blank"} has become the defacto standard for technical documentation, particularly in the Python world. You write your docunment content in [reStructured Text](https://docutils.sourceforge.io/rst.html){:target="_blank"} (RST) markdown ith a simple text editor then build online ready documentation with the [Sphinx](https://www.sphinx-doc.org/en/1.5/tutorial.html){:target="_blank"} compiler.

This article is Part 2 of a 3 parts series where I show you how to use **Read The Docs** to create documentation for a REST API.  [Part 1](){:target="_blank"} covered how to make a basic Sphinx document project. In Part 2, you will add content to document for ficticious API using some of the more sophisticated Sphinx directives.

## Change the Document Theme

The base document you buiit looked nice enough, but was, well, a little plain.  Before launching into the intricacies of Sphinx syntax, let's change the theme to make it look a little more attractive.

If you go to the [Sphinx themes](https://sphinx-themes.org/){:target="_blank"} site, you will see a large number of thmese to choose from. The default theme in the current version of your document is **alabaster**.  It's entirley up to you, and I encourage you to experiment with themes, but the one I like is **sphinx_rtd_theme**, which is in frequent use for Python documentation. Search for this theme, then click on the [sample](https://sphinx-themes.org/html/sphinx_rtd_theme/sphinx_rtd_theme/index.html){:target="_blank"} link to see what a document using the theme looks like.  For the sample document you are building, I'm going to suggest you use the **sphinx_rtd_theme**.

To change the document theme, install the **sphinx_rtd_theme**:

{% highlight bash %}
% cd starterdoc
% pipenv shell
(starterdoc) % pip install sphinx_rtd_theme
{% endhighlight %}

Open `restdoc/source/conf.py` in an editor, then add these lines:

{% highlight html %}
import os, sys
import sphinx_rtd_theme
html_theme = 'sphinx_rtd_theme'
html_theme_path = [sphinx_rtd_theme.get_html_theme_path()]
pygments_style = 'sphinx'
{% endhighlight %}

Build the document then open it in your default browser to see how your document looks.

{% highlight bash %}
(starterdoc) % make clean html
(starterdoc) % open build/html/index.html
{% endhighlight %}

Your document should now look like this:

![First Sphinx Document](/assets/images/First_Sphinx_Document_3.png){:width="100%" .align-center}

## Install HTTP API Markdown

The `sphinxcontrib-httpdomain` module provides that markup that you will use to highlight the API calls and parameters.  Install the module from the top level of your `starterdoc` directory:

{% highlight bash %}
(starterdoc) % pip sphinxcontrib-httpdomain
{% endhighlight %}

To activate the `sphinxcontrib-httpdomain` module, open `restdoc/source/conf.py` in an editor and update the `extensions[]` list to look like this:

{% highlight bash %}
extensions = ['sphinx.ext.autodoc','sphinxcontrib-httpdomain']
{% endhighlight %}


## Add Retrieve API Content


## Add Forms API Content


## Next Steps

