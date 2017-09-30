<h1><p style="text-align: center;">How To Write A Document</p></h1>



Overview
====================
This document is an reference about how to write a document in this repository. 
If you are new to this project and you are editing a document, please keep this document opening.
    
Table of content:
[TOC levels=2-4]

##Create An Document
The first step for creating an document is creating a folder for this document under `src/main/resources/markdown` 
or its sub folder. The name of this folder should reflect the subject of the document and match regular expression
`^[a-z|A-Z|0-9|_]*$`. For example, the folder name for this document is "how_to_write_doc". This folder should
include all material for this document. Such as, markdown files, CSS files and images.

After the folder is created, an markdown document(suffix is ".md") with the same name should be created in this folder.
For example, a "how_to_write_doc.md" file is created for this document. This file is the entry point for the whole 
document. If there is too much content for a document, other "md" files can be created and they can be linked with 
hyperlink. However, this approach is not recommended because the "table of content" `[TOC]` will not be translated
properly.

##How To Use Markdown
###Why Markdown Is Used
Markdown is a lightweight markup language with plain text formatting syntax. Markdown is choose as documentation 
format because:

* The data of markdown document is stored in text format. It is very version control system friendly. Once an version
 control system is used to trace all adjustments, people do not need to use colors to highlight the parts that they
 have adjusted as they did for MS words document.
* Markdown is very simple. It is easy to read and edit. Furthermore, it is simple enough for people to learn the 
  syntax soon. 
* It is different from MS words that user need to use mouse to choose texts and change the format of those texts by 
  clicking some buttons or menu items. In Markdown, user only need to use keyboard to type something to mark the
  texts. This approach can improve the productivity of documentation.
* Markdown is very popular. There are many editors are available in all platform. 
* Markdown can be translated into other format(such as, HTML) easily. That means, it is very easy to distribute 
 documents that are written in Markdown.
 

### Syntax Reference
To illustrate how the rendered result looks like in a document after they are translated by tool 
in this repository, this section show examples for syntax used often. 
Please use this section as a syntax reference when you create a document in this repository.

For more detail information about markdown syntax, please refer [markdown guide](https://www.markdownguide.org/)
 

#### Header

A header can be create as below:
```markdown

Level 1 header
=================

Level 2 header
-----------------

```
The example above looks like:
><h1>Level 1 header</h1>
><h2>Level 2 header</h2>


This approach only can be used to defined first level and second leve headers. A better approach is
```markdown

# Level 1 header

## Level 2 header

### Level 3 header

#### Level 4 header

##### Level 5 header

###### Level 6 header

```
The example above looks like:

><h1>Level 1 header</h1>
>
><h2>Level 2 header</h2>
>
><h3>Level 3 header</h3>
>
><h4>Level 4 header</h4>
>
><h5>Level 5 header</h5>
>
><h6>Level 6 header</h6>


#### Paragraphs
To create paragraphs, use a blank line to separate one or more lines of text. For example:
```markdown
This is the first paragraphs.

This is the second paragraphs.
```
The example above looks like:

>This is the first paragraphs.
>
>This is the second paragraphs.


#### Emphasis
Texts can be made to bold or italic. Below are examples:

| emphasis    |   markdown         | rendered output |
|-------------|--------------------|-----------------|
|  bold       | `**bold**`         | **bold**       |
|  bold       | `__bold__`         | __bold__       | 
|  italic     | `*italic*`         | *italic*        |
|  italic     | `_italic_`         | _italic_        |
|bold & italic| `***italic***`     | ***italic***    |
|bold & italic| `___italic___`     | ___italic___    |
|Strikethrough| `~~Strikethrough~~`|~~Strikethrough~~|


#### Blockquotes
Use `>` to create a blockquote. Below are examples:
```markdown
A blockquote has one paragraphs
> A paragraphs in a blockquote.
This is the second sentence of the paragraphs.

A blockquote has two paragraphs
> Two paragraphs in a blockquote. This is the first paragraphs.
>
> This is the second paragraphs.

```
The example above looks like:
> A blockquote has one paragraphs.
> > A paragraphs in a blockquote.
>   This is the second sentence of the paragraphs.
>   
> A blockquote has two paragraphs.
> > Two paragraphs in a blockquote. This is the first paragraphs.
> >
> > This is the second paragraphs.

####Lists
`*`, `+`, `-` can be used to create unordered list and `numbers` + `.` can be used to creatged ordered list.
For examples:
```markdown
Below is an unordered list:

* item 1
+ item 2
- item 3

Below is an ordered list:

1. item 1
3. item 2
1. item 3
2. item 4
```
The example above is rendered as below:
>Below is an unordered list:
>
>* item 1
>+ item 2
>- item 3
> 
>Below is an ordered list:
>
>1. item 1
>3. item 2
>1. item 3
>2. item 4

#### Codes
` `` ` Can be used to embeded code in one line. Below is an example:
```markdown
Java use `System.out.println()` method to output message.
``` 
The example above will be rendered as:
>Java use `System.out.println()` method to output message.
 
 ` ```{programming language name} ... ``` ` can be used to 
create a fenced code block. This is very useful in case some coding example should be shown in a document. 
For example:



<pre><code>

\```java

@Service
public class UserService {
  ``
  private static final Logger LOG = LoggerFactory.getLogger(UserService.class);

  @Autowired
  UserRepository userRepository;

  @Autowired
  JpaRestPaginationService jpaRestPaginationService;
    
  @ReadOnlyTransaction
      public UserDetailView getUser(String userName) {
      ...
  }
  ...   
}  
\```
</code></pre>

the document will show:
>```java
>@Service
>public class UserService {
>
>    private static final Logger LOG = LoggerFactory.getLogger(UserService.class);
>
>    @Autowired
>    UserRepository userRepository;
>
>    @Autowired
>    JpaRestPaginationService jpaRestPaginationService;
>    
>    @ReadOnlyTransaction
>    public UserDetailView getUser(String userName) {
>        ...
>    }
>    ...   
>}    
>
>``` 

#### Table
Markdown also support table as HTML. Below is an example:
```markdown
|   No alignment     |   Align to right   |   Align to left    |   Align to center    |
|--------------------|-------------------:|:-------------------|:--------------------:|
|   Text             |  Number            |      Date          |2012&#124;12&#124;30  |
|   Text             |  Number            |      Date          |2012&#124;12&#124;29  |
```

The example above is rendered as below:

|   No alignment     |   Align to right   |   Align to left    |   Align to center    |
|--------------------|-------------------:|:-------------------|:--------------------:|
|   Text             |  Number            |      Date          |2012&#124;12&#124;30  |
|   Text             |  Number            |      Date          |2012&#124;12&#124;29  |


### Syntax Exercise
[Markdown Tutorial](https://www.markdowntutorial.com/)

###Config
[vsch maven plugin](https://github.com/vsch/markdown-page-generator-plugin)
