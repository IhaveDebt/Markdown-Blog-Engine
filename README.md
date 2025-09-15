package main

import (
	"flag"
	"fmt"
	"html/template"
	"io/ioutil"
	"os"
	"path/filepath"
	"strings"

	blackfriday "github.com/russross/blackfriday/v2"
)

const layout = `<!doctype html>
<html>
<head><meta charset="utf-8"><meta name="viewport" content="width=device-width"><title>{{.Title}}</title>
<style>body{font-family:system-ui,Arial;margin:2rem;}pre{background:#f5f5f7;padding:1rem;border-radius:6px}</style>
</head><body><h1>{{.Title}}</h1>{{.Body}}</body></html>`

type Page struct {
	Title string
	Body  template.HTML
}

func render(markdown []byte) []byte {
	html := blackfriday.Run(markdown, blackfriday.WithExtensions(blackfriday.CommonExtensions|blackfriday.FencedCode))
	return html
}

func main() {
	src := flag.String("src", "posts", "source directory with .md")
	out := flag.String("out", "public", "output directory")
	page := flag.Bool("page", true, "wrap in layout")
	flag.Parse()

	if _, err := os.Stat(*src); os.IsNotExist(err) {
		fmt.Println("Create a directory named", *src, "and add markdown files.")
		return
	}

	tmpl := template.Must(template.New("layout").Parse(layout))
	_ = os.MkdirAll(*out, 0o755)

	filepath.Walk(*src, func(path string, info os.FileInfo, err error) error {
		if err != nil || info.IsDir() {
			return nil
		}
		ext := strings.ToLower(filepath.Ext(path))
		if ext != ".md" && ext != ".markdown" {
			return nil
		}
		data, _ := ioutil.ReadFile(path)
		html := render(data)
		name := strings.TrimSuffix(filepath.Base(path), ext) + ".html"
		outPath := filepath.Join(*out, name)
		if *page {
			f, _ := os.Create(outPath)
			defer f.Close()
			p := Page{Title: strings.TrimSuffix(filepath.Base(path), ext), Body: template.HTML(html)}
			tmpl.Execute(f, p)
		} else {
			ioutil.WriteFile(outPath, html, 0o644)
		}
		fmt.Println("Generated:", outPath)
		return nil
	})
	fmt.Println("Done. Output in", *out)
}
