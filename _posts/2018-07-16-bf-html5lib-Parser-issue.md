I recently worked on some scraping work with BeautifulSoup for HTML parsing. One interesting finding is that the parser 'html5lib' sometimes is not able to parse the HTML files correctly. For example, when the *li* element goes in the same line as the *ul* element, the parsing would fail. The following example illustrates this.

```python
html1 = '''
<!DOCTYPE html>
<html>
<body>
<div class="page-wrap"><div class="pager">

<ul class="pages"><!----><!----><!----><li class="page-item active"><button class="pagination-btn num-btn">1</button></li>
<li class="page-item"><button class="pagination-btn num-btn">2</button></li>
</ul>
</body>
</html>
'''

bf = BeautifulSoup(html1, 'html5lib')
res = bf.find_all('li')
print(res)
# Return empty list.
```

To solve this parsing issue, just change the parser from 'html5lib' to 'html.parser'.