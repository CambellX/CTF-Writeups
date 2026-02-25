# PDFile
> A friend of mine created this file manager using ChatGPT, and he asked me to find some vulnerabilities but I can’t. Would you like to help me?

## Background
Opening the project, I was met with a file parser of some sort. The fineprint (idk what its called) said it is a ".pasx XML format", but I have no clue what a .pasx file was. So I went straight to the source code given.

```
def parse_pasx(xml_content):
    """Parse .pasx XML content and extract book data."""
    
    if not sanitize(xml_content):
        raise ValueError("XML content contains disallowed keywords.")
    
    try:
        parser = etree.XMLParser(encoding='utf-8', no_network=False, resolve_entities=True, recover=True)
        root = etree.fromstring(xml_content, parser=parser)
        
        book_data = {
            'title': root.findtext('title', default='Untitled'),
            'author': root.findtext('author', default='Unknown Author'),
            'year': root.findtext('year', default=''),
            'isbn': root.findtext('isbn', default=''),
            'chapters': []
        }
        
        chapters = root.find('chapters')
        if chapters is not None:
            for chapter in chapters.findall('chapter'):
                chapter_data = {
                    'number': chapter.get('number', ''),
                    'title': chapter.findtext('title', default=''),
                    'content': chapter.findtext('content', default='')
                }
                book_data['chapters'].append(chapter_data)
        
        return book_data
    except etree.XMLSyntaxError as e:
        raise ValueError(f"Invalid XML: {str(e)}")
```
The program has a simple XML file parser, which immediately made me think of XXE. The program essentially parses an XML tree and looks for a "title", "author", "year", and "isbn" node. 

Testing my idea, I entered a basic XML tree.
```
<root>
<title>Cambell</title>
<author>hey</author>
<year>3065</year>
<isbn>idk what this is</isbn>
</root>
```
The output is:
<body align="left">
  <img src = "images/example.jpg" width=400>
</body>

## Reading the code
The source has a few validation functions. Here they are:
```
def allowed_file(filename):
    return '.' in filename and filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS
```
Makes sure that the user enters a file with the .pasx extension.


```
def sanitize(xml_content):
    try:
        content_str = xml_content.decode('utf-8')
    except UnicodeDecodeError:
        print("error with unicode")
        return False
    
    if "&#" in content_str:
        print("Comment detected")
        return False
    
    blacklist = [
        "flag", "etc", "sh", "bash", 
        "proc", "pascal", "tmp", "env", 
        "bash", "exec", "file", "pascalctf is not fun", # good old censorship
    ]
    if any(a in content_str.lower() for a in blacklist):
        print(a)
        return False
    return True
```
Blacklists a few terms and ensures everything is in

