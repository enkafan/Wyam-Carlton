Title: Basics of the Wyam Blog Recipe
Series: Building a Wyam Theme
Published: 2/8/2017
Tags: Wyam
---

Having reached the important milestone of having one post and then nothing for  months meant it was time to do the next logical step - change the blog engine. I originally built this using [Hugo](https://gohugo.io/), which worked pretty well. But Hugo was written in Go, which meant the barrier for entry for me to contribute back to the project was pretty high, especially for a mature project like Hugo. [Wyam](https://wyam.io/) has gotten more and more press, and with it being written in .NET it seemed like a good opportunity to move to a framework I'm very familiar with plus hopefully contribute back on some low-hanging fruit. 

Wyam comes out of the box with two themes for the blog recipe, but I wanted to use my own because I'm a special snowflake. And it also let me take a deep dive in how everything works. I wanted to write up my journey to build out my own theme. I'll leave a thorough getting started guide to other people - that's been well covered already.

* [Wyam Blog Recipe Overview](https://wyam.io/recipes/blog/overview)
* [Scott Hanselman's "Exploring Wyam"](http://www.hanselman.com/blog/ExploringWyamANETStaticSiteContentGenerator.aspx)

# Creating a New Recipe

Ok, assuming you've installed Wyam and have it on a path lets just build up a quick blog using the blog recipe

``` output
wyam new -r blog
```
What's going on here behind the scenes is pretty simple. The blog recipe is defined in Wyam in [`Blog.cs`](https://github.com/Wyamio/Wyam/blob/develop/src/extensions/Wyam.Blog/Blog.cs). When you call the `new` verb the `Scaffold` method of `Blog` is executed. As of right now the scaffolding will

1. Writes a new config.wyam file with the content of `#recipe Blog`
2. Adds `about.md`, a sample content page
3. Adds `posts/first-post.md`, a sample blog post

That's actuallly all it takes to get started. Running `wyam -p -w` will build your site and launch a server at `http://localhost:5080` using a default theme and settings. Magic! A little too magic for my taste. So let's see what these defaults are doing any why.

# The Blog Recipe

A recipe is simply a predined list of [pipelines](https://wyam.io/docs/concepts/pipelines) and settings. You don't have to use a recipe when using Wyam. Just put all the steps in your config.wyam file and you'll be just fine. The recipes make it quite easy to get started and do the vast majority of the work you need. But I don't think its a good idea to blindly follow them without knowing what's going on under the covers.

The blog recipe's pipelines is defined in the same [`Blog.cs`](https://github.com/Wyamio/Wyam/blob/develop/src/extensions/Wyam.Blog/Blog.cs) that contains the scaffold command. The first part of the file contains a list of settings. These are important! If you want to change how the recipe works without changing the pipeline code then you modify these. You can specify them in the config.wyam file. The obvious two that we'll almost always want to tweak are going to be `Title` and `Description`. You'll notice the code uses `BlogKeys.Title`, `BlogKeys.Description`, etc for the properties. These are merely string contants defined in [`BlogKeys.cs`](https://github.com/Wyamio/Wyam/blob/develop/src/extensions/Wyam.Blog/BlogKeys.cs). So to use the settings from your config file you just need to write

``` csharp
Settings[BlogKeys.Title] = "Phil's Blog";
Settings[BlogKeys.Description] = "It's a blog. By Phil.";
```

Simple enough. If you still have wyam watching your content and save the file you'll notice it rebuilds your project with the new settings. Now onto pipelines. I'd suggest you read through the [pipeline documentation](https://wyam.io/docs/concepts/pipelines) on Wyam.io, but here's the basics of a pipeline

 * A pipeline contains a list of steps defined as `Modules`.
 * Each pipeline typically starts by reading files from disk or the output of a previous pipeline.
 * Each step of a pipeline passes the documents along to modules that can modify them, run Razor, generate metadata about them, etc. Then they pass them along. 
 * The output of a pipeline is the result of all the modules handing the documents step-by-step.
 * A document's properties are defined in [`IDocument`](https://github.com/Wyamio/Wyam/blob/develop/src/core/Wyam.Common/Documents/IDocument.cs) which also inherits from [`IMetadata`](https://github.com/Wyamio/Wyam/blob/develop/src/core/Wyam.Common/Meta/IMetadata.cs). I recommend reading through these two interfaces and learn the properties because they'll be the model for the majority of our layouts.

 ## Pipelines of the Blog Recipe

These are the default pipeline steps of a Blog Recipe. I think it's important to walk through each one because they do make some assumptions that can't be configured like layout pages, how content is laid out, etc. By using the `#recipe Blog` you are more or less importing these pipeline steps into your config file.

### Step 1 - Pages

The first step generates all the pages such as the `about.md` file that was generated as part of the scaffold. These are basically any Markdown or `cshml` file in the `input` folder outside of the `posts` folder. 

1. Read all markdown files outside of the posts file and their front matter.
2. Convert those files from markdown to html.
3. Add all cshtml files outside of the posts path and their front matter.
4. Add `posts/index.cshtml` and its front matter.
5. Add `tags/index.cshtml` and its front matter.
6. Add meta data to the file related to an image and header text color that can be specified in the front matter. If nothing is specified in the file then it adds the default value.
7. Writes all the metadata output to the pipeline. Note the `OnlyMetadata`. This command specifies to add the metadata to the input documents but don't actually write them. We'll use the output of this pipeline later to actually write it to disk.

### Step 2 - RawPosts

This is the step that processes the mark down files. It only reads the files in the posts path (not recursively either you'll notice). At the end of these we won't write the files quite yet either. This will be another step

1. Read files from folder specified in `PostsPath` and their front matter. As of writing I wouldn't change this setting from `posts` myself just because there seems to be other hardcoded dependencies to that folder still.
2. Convert those files from markdown to html .
3. Add any files written in Razor and their front matter.
4. Add meta data indicating whether or not the `Published` key was specified .
5. Add meta data named `Published` that is a `DateTime` that is the parsed `DateTime` value from the front matter.
6. Filter out the documents without a published date or a published date in the past.
7. Add metadata for the `RelativeFilePath. This will be used to generate links to the file.
8. Order these by the published data.

### Step 3 - Tags

This takes the output of the `RawPosts` pipeline and generates new documents based on their tags. This is why step 2 didn't write the files quite yet - we still wanted to process them.

1. Read in `tags/tag.cshtml` and its front matter. Each output tag page will use this as its base.
2. Group the output of the `RawPosts` pipeline by their tags and filter out the ones with no tags. 
3. For each of these documents add the following metadata
    * `Tag` - the tag
    * `Title` - also the tag
    * `Posts` - a list of the documents mapping to that tag
    * `RelativeFilePath` - Name of  file to be written in the tags folder
4. Write each tag file using the `_Layout.cshtml` Razor layout page.

### Step 4 - Posts

Now that we have the tags written, we can write the blog posts

1. Read in the output of `RawPosts`.
2. Apply the razor layout `_PostLayout.cshtml` and render.
3. Adds an Excerpt metadata key with the content of the first `p` tag.
4. Adds metadata named `Content` containing the contents of `div#content`.
5. Write the html.
6. Output the files again, ordered by the `Published` date.

### Step 5 - Feeds

This takes the output of the previous `Posts` pipeline and Writes

1. Rss file.
2. Atom file.
3. Rdf file.

### Step 6 - Render Pages

This takes the output of the `Pages` pipeline and renders them with the `_Layout.cshtml` razor layout

### Step 7 - Redirects

This looks for the `RedirectFrom` front matter and writes files with a meta tag doing the appropriate redirect. Useful for migrating blog engines

### Step 8 - Resources

This copes all non-cshtml and markdown files to the output

### Step 9 - Validate Links 

This can be disabled with the `ValidateAbsoluteLinks` and `ValidateRelativeLinks` settings. If enable it does the following steps

1. Reads the output of `RenderPages`, `Posts` and `Resources` that have an html extensions.
2. Calls the `ValidateLinks` modules and optionally raise an error if the link is invalid.

## Whew

Ok, now that we know what the steps are doing we can approach building our own theme a lot easier. These pipelines define the rules which our theme will have to live. 

1. `posts/index.cshtml` must exist. It has a list of all our posts i.e. the archive page.
2. `tags/index.cshtml` must exist. It has a list of all our tags.
3. `tags/tag.cshtml` must exist. It has the post list for a particular tag.
4. `_layout.cshtml` must exist. It is the default layout page for everything non-post.
5. `_postLayout.cshtml` must exist. It is the layout page for all the posts.
6. `div#content` is what the `Posts` pipeline will use to extract the content of a page. 

The next part of the series I'll take these guidelines and build the simpliest of the simple Wyam themes.