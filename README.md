# Signavio Tech Blog

Rough notes for instructions for bloggers…

## What to write about

* A tutorial for something you learned how to do
* What you wish everyone did differently
* A technical problem you solved
* Fun tech ideas you talked about over beers
* Review/comparison of technologies you chose between

Everyone has something to write about!
See [How To Brainstorm Talk Ideas](http://missgeeky.com/category/geeky-2/).

## Content guidelines

* Keep blog posts short: 400-500 words is plenty, so write another post if you have more to say
* Don’t write ‘this is the first post in a series’ unless the whole series is already _published_
* Write for a developer audience: current and future colleagues, and your future self
* Be nice
* Celebrate work you did, while giving credit where credit is due
* Don’t write anything that would violate our code of conduct
* Don’t make Signavio look bad

## Content schedule

To start with, we’ll publish posts no more frequently than once per week.
A reasonable expectation is that every developer
We should only increase this if we get a backlog of more than four or five unpublished posts.

## How to publish a blog post

1. Pick a ‘short title’ (a.k.a. a slug) to identify the post; you can change this later
1. Pick a planned publication date at least one week after the previous post’s date
1. Create a new branch in the repo, named `post/short-title`
1. Add a file `_posts/yyyy-mm-dd-short-title.md` with the planned publication date
1. Add a [YAML front matter](https://jekyllrb.com/docs/frontmatter/) block, with a `title`, `description` and `tags`
1. Write the blog post in Markdown
1. Build the site by [running Jekyll locally](https://help.github.com/articles/setting-up-your-github-pages-site-locally-with-jekyll/)
1. Commit, push to your branch, and create a pull request
1. Merge to master once you’ve had at least one review and resolved any comments

## Adding images and files to posts

To link to an image, use a relative URL with no subfolders.
Post URL paths start with a year, e.g. `/2017/`, so put image files in the repo’s `/2017` folder.
See existing blog posts for examples.
