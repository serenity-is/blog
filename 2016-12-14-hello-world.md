---
title: Welcome to Serenity Blog
description: It was clear for some time that Serenity.is needed its own blog where i could share some news, tips and so on with the community. After some evaluation and planning, I decided to develop a quick and dirty one for myself. Here is the story and step by step how-to.
author: Volkan Ceylan
---

## What Makes a Basic Blog Post?

A blog post should have an *Author*, *Title*, *Description* (excerpt), *Date Posted* and *Post Body* at minumum.

Even though there are some blogs which use *HTML* for the article body, my favorite as a developer has always been Markdown, so i decided to go with that.

If i was developing this for some end user, it would be useful to offer an HTML editor option as they don't like Markdown much and prefer a MS Word like interface provided by CKEditor and similar.

## Converting Markdown to HTML

There are so many Markdown converters out there, both client side and server side, written with Javascript, PHP, Ruby and many other languages. After some research in GitHub, i decided to go
with Markdig (https://github.com/lunet-io/markdig). It looks like it's fast, supports every extension i could dream of, and is possible to inject custom handlers, if i'll need it.

It has a NuGet package, so you just install it:

```
Install-Package Markdig
```

Then it's a one liner to convert basic Markdown to HTML:

```cs
Markdown.ToHtml("This is a text with some *emphasis*")
```


## To Database or Not to Database

Current trend for sites like Blogs, Documentation etc. is towards static site generators. GitHub Pages uses Jekyll, probably one of the most favorite ones. I use GitBook for Serenity Guide.

There are probably thousands more, Hugo, Hexo, Gatsby, Pelican to name a few. Here is a site with some open source options:

https://www.staticgen.com/

They are said to offer some advantages like simplicity, speed, resources, security etc. I'm not a fan yet, but i decided to go with something close to them.

It would be very simple to create a table and generate code for it using Sergen, then add a few attributes and have a basic interface to edit a blog.

My experience with GitBook so far (for writing documentation) has been mostly positive. I like the idea of using Markdown / Git couple for quick writing and versioning.

So instead of a database, my choice was to design a simple, file based blog.

I added a BlogPosts folder under my SerenityIs.Web project (yes Serenity.is is also a project created from Serene template), and it currently contains this file:

```txt
Serenity.Web\
	BlogPosts\
		2016-12-14-hello-world.md
```

It contains the blog article you are currently reading, egg or chicken ;)

Clearly, first part of the file name, *2016-12-14* contains the post date in yyyy-MM-dd format, so that my blog posts will be in natural order.

Remaining part of the file name, up to *.md* extension, is the dasherized article URL. You are reading this post at URL:

https://serenity.is/blog/2016/12/14/hello-world


But of course, there is no such *blog* folder, nor the following folders or files in my application. That URL pattern is handled by a controller (*BlogController*) with a special route attribute. We'll come to that later.

Point is, to add a blog post, i can add a new MarkDown (.md) file to my *BlogPosts* folder, commit, push to my private GitHub repository, and it will automatically be shown in Serenity.is blogs area.

> I have a very simple automated publish script under Serenity.is server, that pulls the latest changes from GitHub, builds the project, copies required files to production site etc. I'll share code for that in another post.

Another, much simpler option is to directly create/edit the file online at GitHub.

When i'll be adding a blog module to Serene, i'll probably have to offer a database option, but for now this does the job.


## Adding a Blog Controller

To view blog posts, we first need a Blog controller:

```cs
[RoutePrefix("Blog"), Route("{action=index}")]
public class BlogController : Controller
{
}
```

Nothing special so far. As RoutePrefix for this controller is *Blog*, it will handle URLs that start with */Blog* by default.

Time to add an action that will display blog posts.

## Blog Post action

private static BlogFileWatcher watcher;

```cs
private static Regex Hypenated = new Regex("^([\\w-]+)$");

[Route("{year:int:length(4)}/{month:int:length(2)}/{day:int:length(2)}/{*pathInfo}")]
public ActionResult Post(int year, int month, int day, string pathInfo)
{
    if (!Hypenated.IsMatch(pathInfo))
        return new HttpNotFoundResult();

    var key = year.ToString() + "-" + month.ToString("00") + "-" + day.ToString("00") + "-" + pathInfo;
    var post = GetBlogPost(key);
	if (post == null)
		return new HttpNotFoundResult();

    return View(MVC.Views.Frontend.Blog.BlogPost, post);
}
```

Let's dissect this action starting by its route. It will handle URLs of pattern *year/month/day/dasherized-article*,
but as its controller has a routeprefix, it will actually be */blog/year/month/day/dasherized-article*.

We specify some constraints for year, month, day parameters. They all need to be of integer type (e.g. digits) and
have a constant length (4 or 2).

We call the remaining part of URL that will contain dasherized article key as *pathInfo*.

It's possible to also set constraint for last parameter using a regex string, but i choose to check it manually.

*Hypenated* is a regex which will ensure that pathInfo will only contain letters, digits and dashes. If it is not, i return a HTTP 404.

My blog article key (some kind of ID) is a filename without extension, like `2016-12-14-hello-world`. Thus, i take action parameters and concat them with dashes to get the actual article key.

Then i call my helper method to get a post by its key (*GetBlogPost*).

Finally, return my BlogPost view with *post* object as its model.

## BlogPost Class

This is a simple type that will hold information about an article.

```
public class BlogPost
{
    public DateTime Date { get; set; }
    public string Title { get; set; }
    public string Description { get; set; }
    public string Author { get; set; }
    public string[] Categories { get; set; }
    public string HtmlContent { get; set; }
}
```

We use this as a model for our BlogPost view.

OK, but we only have a file name. We might get Date information from its first 10 characters. Where do we get these other information from?

## Storing Metadata For Blog Posts

As we are not using any database, the most natural choice to store metadata about an article is inside the markdown file itself.

The article you are currently reading starts with this content:

```md
---
title: Welcome to Serenity Blog
description: It was clear for some time that Serenity.is needed its own blog where i could share some news, tips and so on with the community. After some evaluation and planning, I decided to develop a quick and dirty one for myself. Here is the story and step by step how-to.
author: Volkan Ceylan
---

## What Makes a Basic Blog Post?

A blog post should have a *Author*, *Title*, *Description* (excerpt), *Date Posted* and *Post Body* at minumum.
...
...
```

Metadata is marked with three dashes (`---`). I am parsing title, description, and author metadata from this MarkDown file with a very basic (and lame) parser which we will see soon.

## GetBlogPost Method

This is the actual method that will load a blog post, given its key, from corresponding Markdown file, parse metadata, and return a BlogPost object:

```cs
private BlogPost GetBlogPost(string key)
{
    if (string.IsNullOrEmpty(key) || !Hypenated.IsMatch(key))
        return null;

    key = key.ToLowerInvariant();

    return LocalCache.Get("BlogPost:" + key, TimeSpan.Zero, () =>
    {
		// ... we'll dissect here soon
	});
}
```

I'm a bit paranoid about security, so i check here that key is valid dasherized string, containing no dangerous characters.

Then, we are ignoring case by lowercasing the key.

It should be enough to parse a blog post once, and cache the result, as long as source file doesn't change. So, i'm using a
LocalCache.Get call here, to cache blog posts by their keys infinitely.

Let's enter the part between LocalCache.Get braces:

```cs
var root = Server.MapPath("~/BlogPosts/");

var path = root + key + ".md";
if (!System.IO.File.Exists(path))
    return (BlogPost)null;

if (key.Length < 11)
    return null;

DateTime date;
if (!DateTime.TryParseExact(key.Substring(0, 10), "yyyy-MM-dd",
        CultureInfo.InvariantCulture, DateTimeStyles.None, out date))
    return null;

var post = new BlogPost();
post.Date = date;
post.Title = key.Substring(11);

```

As i said, i'm storing posts under a `BlogPosts` folder, we use Server.MapPath to get its absolute path.

Then we check if there is actually such a file with given key.

To be sure that it starts with the date for article, we use DateTime.TryParseExact.

Next, we create the BlogPost object and populate its basic information, e.g. Date and default title extracted from the file name.

## Extracting Metadata from MarkDown File

Here comes my basic parser for metadata:

```cs
var lines = System.IO.File.ReadAllLines(path).ToList();
var dashes = lines.FindIndex(x => x.IsTrimmedSame("---"));
if (dashes < 0 || (lines.Take(dashes).Any(x => !x.IsTrimmedEmpty())))
    return null;

var endDashes = lines.FindIndex(dashes + 1, x => x.IsTrimmedSame("---"));
if (endDashes < 0)
    return null;

for (var i = dashes + 1; i < endDashes; i++)
{
    var parts = lines[i].Split(new char[] { ':' }, 2);
    if (parts.Length != 2)
        continue;

    var k = parts[0].TrimToEmpty().ToLowerInvariant();
    var v = parts[1].TrimToEmpty();
    switch (k)
    {
        case "title":
            post.Title = v;
            break;
        case "description":
            post.Description = v;
            break;
        case "author":
            post.Author = v;
            break;
        case "categories":
            post.Categories = v.Split(new char[] { ',', ';' }, StringSplitOptions.RemoveEmptyEntries);
            break;
    }
}
```

Simply, we find index of first three dash line, and last one. Also making sure that no content comes before first three dash line.

Between the three dash lines, every line contains one metadata. So using a switch / case, we fill in those properties.

> I really don't like this parser, in case you didn't notice.

## Converting Markdown to HTML

I'm simply using Markdig, enabling its extensions by creating a pipeline option.

```
var markdown = string.Join("\n", lines.Skip(endDashes + 1));
var pipeline = new MarkdownPipelineBuilder().UseAdvancedExtensions().Build();
var htmlContent = Markdown.ToHtml(markdown, pipeline);
post.HtmlContent = htmlContent;
```


## Validating cache

As we are caching blog posts infinitely, if a post file changes, our article will still display old version.

So need a way to validate cached posts. We'll make use of a FileSystemWatcher:

```
		var htmlContent = Markdown.ToHtml(markdown, pipeline);
		post.HtmlContent = htmlContent;

		watcher = watcher ?? new BlogFileWatcher(root);

		return post;
```

We use a singleton, static file system watcher to validate any blog post:

```
private static BlogFileWatcher watcher;

public class BlogFileWatcher
{
    public BlogFileWatcher(string path)
    {
        var sw = new FileSystemWatcher(path, "*.md");
        sw.IncludeSubdirectories = false;
        sw.NotifyFilter = NotifyFilters.FileName | NotifyFilters.LastWrite;
        sw.Changed += (s, e) => Changed(e.Name);
        sw.Created += (s, e) => Changed(e.Name);
        sw.Deleted += (s, e) => Changed(e.Name);
        sw.Renamed += (s, e) => Changed(e.OldName);
        sw.EnableRaisingEvents = true;
    }

    private void Changed(string name)
    {
        if (!name.EndsWith(".md", StringComparison.OrdinalIgnoreCase))
            return;

        name = name.Substring(0, name.Length - ".md".Length).ToLowerInvariant();
        LocalCache.Remove("BlogPost:" + name);
		LocalCache.Remove("BlogPosts");
    }
}
```

We are watching any change to any file under *BlogPosts* folder, extracting file name from changed path, then removing that article from cache, so that it will be loaded and parsed again.

## BlogPost View

Here is the code for BlogPost.cshtml file:

```html
@model SerenityIs.Frontend.Pages.BlogPost

@{
    ViewData["Title"] = "Blog";
    ViewData["PageId"] = "Blog";
}

@section Head {
<style type="text/css">
    #s-BlogPage .blog-header { padding: 40px 0; }
    #s-BlogPage .blog-header .author-date { font-size: 18px; }
    #s-BlogPage .blog-header h1 { font-size: 48px; }
    #s-BlogPage .blog-header .description { margin: 30px 0 0 5px; font-size: 24px; }
    #s-BlogPage .blog-content { min-height: 600px; }
    #s-BlogPage .blog-content h2 { margin-top: 50px; }
    #s-BlogPage .blog-content pre code { font-size: 16px; }
    #s-BlogPage .blog-content pre { margin-bottom: 25px; }
</style>
<link href="~/Content/highlightjs/monokai-sublime.css" rel="stylesheet" />
<script src="~/Scripts/highlight.pack.js"></script>
<script>hljs.initHighlightingOnLoad();</script>
}

<div class="blog-header">
    <p class="author-date">
        <span class="date">@Model.Date.ToString("MMM dd, yyyy")</span><span> - </span><span class="author">@Model.Author</span>
    </p>
    <h1>@Model.Title</h1>
    <p class="description">@Model.Description</p>
</div>
<hl class="dashed"></hl>
<div class="blog-content">
    @Html.Raw(Model.HtmlContent)
</div>
```

## Highlighting Code Blocks

To highlight code samples in this article, i'm making use of very simple and nice highlight.js library:

https://highlightjs.org/

It's just about including theme CSS file and highlight.pack.js in page and calling:

```
<script>hljs.initHighlightingOnLoad();</script>
```

Like magic.

## That's All Folks!

It took much more time to write this article than developing my basic blog.

There might be some steps i missed, like the listing page. I'll update this article later.

See you with the next one.