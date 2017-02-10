Title: Scaffolding our Wyam Theme
Series: Building a Wyam Theme
Published: 2/9/2017
Tags: Wyam
---
Let's recap the guidelines we found while [examing the Blog Recipe](/posts/wyam-theme-part-1.html)

1. `posts/index.cshtml` must exist. It has a list of all our posts i.e. the archive page.
2. `tags/index.cshtml` must exist. It has a list of all our tags.
3. `tags/tag.cshtml` must exist. It has the post list for a particular tag.
4. `_layout.cshtml` must exist. It is the default layout page for everything non-post.
5. `_postLayout.cshtml` must exist. It is the layout page for all the posts.
6. `div#content` is what the `Posts` pipeline will use to extract the content of a page. 

Based on these rules lets create the simpliest of themes. I'm going to store all the theme files in their own folder. - I'll name it `barebones` for now.  To keep things simple I'm going to dump all the code in `_layout.cshtml` for rendering the full page. `_postLayout.cshtml` will defer to it and just contain the simplest of code. 

``` csharp
@{
    Layout = "_Layout.cshtml";
}

@RenderBody()
```

## Getting the Current Page's Metadata

We'll use this layout page for everything. The homepage, archive, tags and individual posts. We'll need to be slightly careful to not be using data that might not exist depending on the context. But luckily `IMetadata` has some helper methods that will make life easier on us. 

So at the top of the page we want to grab some page's data and store it for easier access. Let's tackle a blog post first. For that we'll want to grab some specific  meta datafrom the current document to render on the page. Specifically we want the published date and a list of tags. Let's put a script block at the top of the page. We can then grab that data using the following code

``` csharp
@{
    var tags = Model.Get<string[]>(BlogKeys.Tags);
    var published = Model.Get<DateTime>(BlogKeys.Published);   
}
```

What's nice about these helpers is that if the value doesn't exist it'll return `null` rather than throwing an exception. That's important for pages like the homepage or the about page which won't have the tags or published metadata set. The other piece of data that we'll want is the current page's title. We can do that pretty simply by adding the following code to our code block defined above

``` csharp
var title = Model.WithoutSettings.String(BlogKeys.Title);
```

Notice the use of `WithoutSettings`. This is important because typically a call to `Model.String` would default back to the global setting in our `config.wyam` file. We only want to know if the current page has a title here, so we want a null value if the current page doesn't have a title defined in it's front matter.

But there will be cases where we'll need to display something as the title, such as the homepage's `title` element. In those cases we will want the default value. So we add the following code to account for this scenario

``` csharp
var pageTitle = Context.String(BlogKeys.Title) + (string.IsNullOrWhiteSpace(title) ? string.Empty : " - " + title);
```

In this case we'll use `Context.String` to pull the value from the config to get the site title. Then if we pulled a title from the page's front matter we'll append that. Perfect for a `title` element.

So lets use `pageTitle`. Our `_layout.cshtml` should now look something like this

``` html
@{
    var title = Model.WithoutSettings.String(BlogKeys.Title);
    var pageTitle = Context.String(BlogKeys.Title) + (string.IsNullOrWhiteSpace(title) ? string.Empty : " - " + title);

    var tags = Model.Get<string[]>(BlogKeys.Tags);
    var published = Model.Get<DateTime>(BlogKeys.Published);   
}

<!DOCTYPE html>
<html lang="en">
<head>
    <title>@pageTitle</title>
</head>
```

## Rendering the Page Header

Ok, let's actually get something on the screen. Let's start with the page header. We'll keep it simple and put the `title` we pulled above in a `h1` block, followed by some child page links. To create the links, we'll use the helper method `Context.GetLink`. This generates a normalized link. We don't really need it here, but it can't hurt.

So let's add the following html to our layout

``` html
<h1><a href="/">@Context.String(BlogKeys.Title)</a></h1>
<ul>
    <li><a href="@Context.GetLink("/about")">About Me</a></li>
    <li><a href="@Context.GetLink("/posts")">Archive</a></li>
    <li><a href="@Context.GetLink("/tags")">Tags</a></li>
</ul>
```

With our site title in place, now we can tackle the individual page title and header. We'll build this up using the metadata we pulled above. Because this might not exist we'll need some conditionals statements to make sure things look ok, but that's no big deal.

``` html
@if (string.IsNullOrWhiteSpace(title) == false)
{
    <h1><a href="@Context.GetLink(Model)">@title</a></h1>
}

<div class="meta">
    @if (published != default(DateTime))
    {
        <span>@published.ToLongDateString()</span>
    }
</div>
```

We still have the tags, but we'll add those in a bit. Last thing we need to do is call `@RenderBody` and close up the `body` and `html` tags and we should be done. But don't forget - the the content is expected to be in a `div#content` tag to pull the appropriate metadata.

Our full and barebones layout is now ready to be applied. To do this we can either run `wyam -p -w -t barebones` or we can configure it in the `config.wyam` 

``` output
#theme barebones
```

At this point you should have at least something being generated. Assuming you still have your `first-post.md` you should be able to navigate to http://localhost:5080/posts/first-post.html` and view your header and the content of the file. Neat. Still work to do obviously.

## Creating the Home Page

Ok, let's make the homepage. This will be `index.cshtml` in the root of our barebones folder. What we want to do is dump the full content of the first five posts on our blog. To do that it's quite simple - we use the `Documents` dictionary to access the output of one the Posts pipelines. The syntax to get the full list is simply `Documents[BlogPipelines.Posts]`. Also remember that each of these documents has a metadata field named `Content` that contains everything in the `div#content` tag of that page. So all we need to do is loop through the first five documents and output their contents. This makes our `index.cshtml` look something like this

``` csharp
foreach(IDocument doc in Documents[BlogPipelines.Posts].Take(5))
{
    @Html.Raw(doc.String(BlogKeys.Content))
}
```

Assuming you left wyam running, if you save this file you should see it detect the change and create the new index.html page in the root. Navigate to http://localhost:5080 and you should see the first 5 posts. In our case it'll just be one, but you can dump some extra dummy content in there to verify if you'd like.

## Creating the Archive Page

This should list all of our posts, grouped by year. Again, we'll use `Documents[BlogPipelines.Posts]` to pull all the documents. In this case also group them by the year they are published. So two nested loops should work fine here. Our archive page ends up looking like

``` csharp
Title: Archive
---
@{
    Layout = "/_PostLayout.cshtml";
}

@{
    foreach(IGrouping<int, IDocument> year in Documents[BlogPipelines.Posts]
        .GroupBy(x => x.Get<DateTime>("Published").Year)
        .OrderByDescending(x => x.Key))
    {
        <h1>@year.Key</h1>
        <ul>
            @foreach(var document in year.OrderByDescending(x => x.DateTime("Published")))
            {
                <li><a role="button" href="@Context.GetLink(document)">@document.String(BlogKeys.Title)</a> @document.DateTime("Published").ToString("MMM d")</li>
            }
        </ul>
    }
}
```

### Create the Tag pages

Create the tag related pages will be very similiar to how the archive page worked. But instead of using the output from the `Posts` pipeline, we'll use the output from the `Tags` pipeline. Remember this generated a list of documents, one for each tag. So we need to merely loop through these documents. 

The basic tags `index.cshtml` page will look something like this

``` html
Title: All Tags
---

@{
    Layout = "_Layout.cshtml";
}

<ul>
    @foreach (IDocument tagDocument in Documents[BlogPipelines.Tags].OrderBy(x => x.String(BlogKeys.Tag)))
    {
        string tag = tagDocument.String(BlogKeys.Tag);
        string postCount = tagDocument.DocumentList(BlogKeys.Posts).Count().ToString();
        <li><a role="button" href="@Context.GetLink(tagDocument)"> @tag (@postCount)</a></li>
    }
</ul>
```

You'll notice we are reaching into the tag document's `DocumentList` metadata to pull the count of documents related to that tag.

Now we need the actual tag page. Open up `tag.cshtml`. This document is even simpler. Here we looping through the tag's `DocumentList` metadata ordered by the `Published` date and writing links to them. We have full access to all those document metadata too, so we could easily add excerpts or the published date to the output too.

``` html
@{
    Layout = "/_PostLayout.cshtml";
}

<ul>
    @foreach (IDocument document in Model.DocumentList(BlogKeys.Posts).OrderByDescending(x => x.Get<DateTime>(BlogKeys.Published)))
    {
        <li><a role="button" href="@Context.GetLink(document)">@document.String(BlogKeys.Title)</a></li>
    }
</ul>
```

At this point, we have ourselves a full blog with a layout Tim Berners Lee would be proud of 