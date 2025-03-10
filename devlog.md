
# Future improvements 

  * Google Analytics
  * _site/assets/img/favicon - confirm needed (for apple devices?), resize main logo (once settled on what it will be)
  * Consider whether to add email address and/or set up a dedicated email address for the blog

# Historical notes

## Initial setup (10/3/25)

* Download Ruby and Devkit - for local testing
  - Windows installer https://rubyinstaller.org/downloads/
  - Include the ~1GB devkit
  - Use the x64 version (right click Windows icon > System to confirm system type)

* Create a new repo
  - https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-github-pages-site - created repo at <user>/<user>.github.io
  - must be public (for free Github Pages usage)
  - include the optional description (ChatGPT useful for ideation)
  - MIT licence (permissive)

* Find a template
  - https://github.com/artemsheludko/flexible-jekyll - clean (and free)
  - downloaded files rather than forking (inclined to customise it in the future)

* Edit files in repo
  * _config.yml (title, description, author settings)
  * README.md
  * add devlog.md for tracking steps taken (possible future blog post), place to park ideas for the future

* Test locally (in vscode Terminal, click server address link in terminal outputs to open site in browser window)

```
bundle install
bundle exec jekyll serve
```

* Made further changes in response to local testing
  * posts in the _posts directory that don't have yyyy-mm-dd- before the post name won't be included in the build, so don't need to have a separate drafts folder, but do need to add a date when testing locally or it won't display
  * adjusted code in main.html within the else conditions as it was defaulting to the creator's social media handles if you leave values blank in, also reordered so LinkedIn > GitHub > Email
  * _config.yml rather than just removing unused ones (e.g. not having a link to Facebook website)
  * used https://favicon.io/favicon-converter/ to create an icon file which shows in the browser tab
  * confirmed you can use folders within img (e.g. place all images for a given post in a matching folder so things are organised)
  * confirmed you can use .md rather than .markdown as _posts file extensions
  * decided to include my personal email in the site code (taking on the risk of scrapers and spam, at the potential benefit of being more easily contactable)
    * alternatives considered include contact form requiring more complex website template + form service (can be free at low volumes), not having this means of contact (can rely on LinkedIn or simply be less contactable)

* Random research to understand the Jekyll parameters (e.g. in front matter, file name)
  * The description field in the front matter (at the top of posts) is likely used for metadata descriptions. These are used (or not used as it turns out -  [this page](https://converted.co.uk/7-reasons-why-your-meta-title-meta-description-arent-showing-in-google/) indicates Google rewrites it 70% of the time) as a summary shown by search engines under the link, which may entice readers. This explains why I'm writing a description that seems to not appear in my post, which is probably pretty similar to my title and/or introduction
  * The date field in the front matter can differ from the date used in the file path. The file path is used for sorting posts and generating permalinks (when using default Jekyll settings). Publish dates are based on the front matter; according to ChatGPT future dates aren't published unless future: true is included in _config.yml, from testing this is indeed the case (noting this applies to the front matter date and not the filename date). The front matter date may also be used for RSS feeds.

* Created a few simple posts mainly for testing purposes
  * images need to be in a wide perspective, not square perspective, manually trim to XxX pixels
    * ~293 x 179 (1.64:1) with my browser maximised or ~224x177 (1.27:1) if I reduce browser width significantly
    * for a 1024x1024 square image, reducing to 1024x620 (with 620=1024/1.65) gives a nice displayed image
    * the sizes above were on the home page; in the actual posts, the header image was ~1000x500, so 2:1
    

* Decided on a branch strategy - using the simpler approach of having any changes to main lead to an update to the site, then using a dev branch if I want to make coding changes (and potentially test/develop them across different devices over time without updating the site)
  - this is in contrast to the use of a gh-pages branch, which seems to be more common for larger projects/teams, custom builds, using alternative CI/CD frameworks etc

* Initialise local repo

```
git init # initialise Git tracking locally
git add . # stage all files for commit
git commit -m "Initial commit" # save current state
git branch -M main # rename default branch to main
git remote add origin <your-repo-url>
git push -u origin main # upload files to GitHub
```

* https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site - 