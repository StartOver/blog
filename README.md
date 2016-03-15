# My Pelican Blog


## Usage


```
$ git clone git@github.com:startover/blog.git
$ cd blog
$ pip install pelican markdown  # install dependency
$ fab build  # generate the .md files to output folder
$ fab serve  # start server bind to localhost:8000 
```

## Publish

```
$ pip install ghp-import
$ make github
```

