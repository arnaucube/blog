# Static blog template engine implementation in Go
*2017-12-26*

Some days ago, I decided to start this blog, to put there all the thoughts and ideas that goes through my mind. After some research, I've found some interesting projects, but with a lot of features that I don't need to use. So I decided to write my own minimalistic static blog template engine with Go lang.

This is how I made [blogo](https://github.com/arnaucube/blogo) the blog static template engine to do this blog.


### Static blog template engine?
The main idea is to be able to write the blog posts in Markdown files, and with the template engine, output the HTML files ready to upload in the web hosting server.

## Structure
What we wan to have in the input is:

```
/blogo
    /input
        /css
            mycss.css
        /img
            post01_image.png
        index.html
        postThumbTemplate.html
        post01.md
        post01thumb.md
        post02.md
        post02thumb.md
```

#### blogo.json
This is the file that have the configuration that will be read by the Go script.

```json
{
    "title": "my blog",
    "indexTemplate": "index.html",
    "postThumbTemplate": "postThumbTemplate.html",
    "posts": [
        {
            "thumb": "post01thumb.md",
            "md": "post01.md"
        },
        {
            "thumb": "post02thumb.md",
            "md": "post02.md"
        }
    ],
    "copyRaw": [
      "css",
      "js"
    ]
}
```
The *copyRaw* element, will be all the directories to copy raw to the output.

#### index.html
This is the file that will be used as the template for the main page and also for the posts pages.

```html
<!DOCTYPE html>
<html>
<head>
    <title>[blogo-title]</title>
</head>
<body>
    <div>
        [blogo-content]
    </div>
</body>
</html>
```

As we can see, we just need to define the html file, and in the title define the *[blogo-title]*, and in the content place the *[blogo-content]*.

#### postThumbTemplate.html
This is the file where is placed the html box for each post that will be displayed in the main page.

```html
<div class="well">
    [blogo-index-post-template]
</div>
```

#### post01thumb.md

```markdown
# Post 01 thumb
This is the description of the Post 01
```

#### post01.md
```markdown
# Title of the Post 01
Hi, this is the content of the post

'''js
    console.log("hello world");
'''
```

## Let's start to code
As we have exposed, we want to develop a Go lang script that from some HTML template and the Markdown text files, generates the complete blog with the main page and all the posts.

#### readConfig.go
This is the file that reads the *blogo.json* file to get the blog configuration.

```go
package main

import (
	"encoding/json"
	"io/ioutil"
)

//Post is the struct for each post of the blog
type Post struct {
	Thumb string `json:"thumb"`
	Md    string `json:"md"`
}

//Config gets the config.json file into struct
type Config struct {
	Title             string   `json:"title"`
	IndexTemplate     string   `json:"indexTemplate"`
	PostThumbTemplate string   `json:"postThumbTemplate"`
	Posts             []Post   `json:"posts"`
	CopyRaw           []string `json:"copyRaw"`
}

var config Config

func readConfig(path string) {
	file, err := ioutil.ReadFile(path)
	check(err)
	content := string(file)
	json.Unmarshal([]byte(content), &config)
}
```

#### files.go, the operations with files
We will need some file operation functions, so we have placed all in this file.

```go
package main

import (
	"io/ioutil"
	"os/exec"
	"strings"

	"github.com/fatih/color"
)

func readFile(path string) string {
	dat, err := ioutil.ReadFile(path)
	if err != nil {
		color.Red(path)
	}
	check(err)
	return string(dat)
}

func writeFile(path string, newContent string) {
	err := ioutil.WriteFile(path, []byte(newContent), 0644)
	check(err)
	color.Green(path)
}

func getLines(text string) []string {
	lines := strings.Split(text, "\n")
	return lines
}

func concatStringsWithJumps(lines []string) string {
	var r string
	for _, l := range lines {
		r = r + l + "\n"
	}
	return r
}

func copyRaw(original string, destination string) {
	color.Green(original + " --> to --> " + destination)
	_, err := exec.Command("cp", "-rf", original, destination).Output()
	check(err)
}
```

#### main.go

To convert the HTML content to Markdown content, we will use https://github.com/russross/blackfriday

```go
package main

import (
	"fmt"
	"strings"

	blackfriday "gopkg.in/russross/blackfriday.v2"
)

const directory = "blogo-input"

func main() {
	readConfig(directory + "/blogo.json")
	fmt.Println(config)

	// generate index page
	indexTemplate := readFile(directory + "/" + config.IndexTemplate)
	indexPostTemplate := readFile(directory + "/" + config.PostThumbTemplate)
	var blogoIndex string
	blogoIndex = ""
	for _, post := range config.Posts {
		mdpostthumb := readFile(directory + "/" + post.Thumb)
		htmlpostthumb := string(blackfriday.Run([]byte(mdpostthumb)))

		//put the htmlpostthumb in the blogo-index-post-template
		m := make(map[string]string)
		m["[blogo-index-post-template]"] = htmlpostthumb
		r := putHTMLToTemplate(indexPostTemplate, m)
		filename := strings.Split(post.Md, ".")[0]
		r = "<a href='" + filename + ".html'>" + r + "</a>"
		blogoIndex = blogoIndex + r
	}
	//put the blogoIndex in the index.html
	m := make(map[string]string)
	m["[blogo-title]"] = config.Title
	m["[blogo-content]"] = blogoIndex
	r := putHTMLToTemplate(indexTemplate, m)
	writeFile("index.html", r)

	// generate posts pages
	for _, post := range config.Posts {
		mdcontent := readFile(directory + "/" + post.Md)
		htmlcontent := string(blackfriday.Run([]byte(mdcontent)))

		m := make(map[string]string)
		m["[blogo-title]"] = config.Title
		m["[blogo-content]"] = htmlcontent

		r := putHTMLToTemplate(indexTemplate, m)
		//fmt.Println(r)

		filename := strings.Split(post.Md, ".")[0]
		writeFile(filename+".html", r)
	}

	//copy raw
	fmt.Println("copying raw:")
	for _, dir := range config.CopyRaw {
		copyRaw(directory+"/"+dir, ".")
	}
}

func putHTMLToTemplate(template string, m map[string]string) string {
	lines := getLines(template)
	var resultL []string
	for _, line := range lines {
		inserted := false
		for k, v := range m {
			if strings.Contains(line, k) {
				//in the line, change [tag] with the content
				lineReplaced := strings.Replace(line, k, v, -1)
				resultL = append(resultL, lineReplaced)
				inserted = true
			}
		}
		if inserted == false {
			resultL = append(resultL, line)
		}
	}
	result := concatStringsWithJumps(resultL)
	return result
}
```

## Try it
To try it, we need to compile the Go code:
```
> go build
```
And run it:
```
> ./blogo
```

Or we can just build and run to test:
```
> go run *.go
```

As the output, we will obtain the html pages with the content:

- index.html

```html
<!DOCTYPE html>
<html>
<head>
    <title>my blog</title>
</head>
<body>
    <div class="row">
        <a href='post01.html'>
            <div class="col-md-3">
                <h1>Post 01 thumb</h1>
                <p>This is the description of the Post 01</p>
            </div>
        </a>
        <a href='post02.html'>
            <div class="col-md-3">
                <p>Post 02 thumb</p>
            </div>
        </a>
    </div>
</body>
</html>
```

- post01.html

```html
<!DOCTYPE html>
<html>
<head>
    <title>my blog</title>
</head>
<body>
    <div>
        <h1>Title of the Post 01</h1>
        <p>Hi, this is the content of the post</p>
        <pre>
            <code class="language-js">    console.log(&quot;hello world&quot;);
            </code>
        </pre>
    </div>
</body>
</html>
```

## Conclusion
In this post, we have seen how to develop a very minimalistic static blog template engine with Go. In fact, is the blog engine that I'm using for this blog.

There are lots of blog template engines nowadays, but maybe we don't need sophisticated engine, and we just need a minimalistic one. In that case, we have seen how to develop one.

The complete project code is able in https://github.com/arnaucube/blogo

