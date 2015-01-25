Pygments styles generated using the following command line

pygmentize -f html -S monokai -a .highlight > syntax.css 

To enable horizontal scrolling add the following at the beginning of the syntax.css file
.highlight pre {
    white-space: pre;
    overflow: auto;
    word-wrap: normal;
}
