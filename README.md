# Before setup

Install Hugo if it is not installed.

```bash
# macOS
brew install hugo
```

```bash
# Ubuntu, Linux that supported "snap"
snap install hugo
```

# Setup

Clone the project.

```bash
git clone git@github.com:infaspublications/blog.git
cd blog
```

Start Hugo and preview

```bash
cd infaspublications.github.io
hugo server
```

Please open http://localhost:1313/ in your browser.

# Write a new post

```bash
cd infaspublications.github.io
hugo server --watch
hugo new post/awesome-post.md
```

You can add posts by editing `infaspublications.github.io/content/post/awesome-post.md`.

# Publish new posts

```bash
cd infaspublications.github.io
hugo
cd public
git add -A
git commt -m 'New post: Awesome title'
git push origin master
cd ../../ # Go to root of "blog" project
git add -A
git commt -m 'New post: Awesome title'
git push origin master
```
