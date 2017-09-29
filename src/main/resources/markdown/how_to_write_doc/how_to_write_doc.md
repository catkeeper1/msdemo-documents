<h1><p style="text-align: center;">How To Write A Document</p></h1>



Overview
====================
This document is an reference about how to write a document in this repository. 
If you are new to this project and you are editing a document, please keep this document opening.
    
Table of content:
[TOC]

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
###Why "Markdown" Is Used
"Markdown" is a lightweight markup language with plain text formatting syntax. "Markdown" is choose as documentation 
format because:

* The data of markdown document is stored in text format. It is very version control system friendly. Once an version
 control system is used to trace all adjustments, people do not need to use colors to highlight the parts that they
 have adjusted as they did for MS words document.
* "Markdown" is very simple. It is easy to read and edit. Furthermore, it is simple enough for people to learn the 
  syntax soon. 
* It is different from MS words that user need to use mouse to choose texts and change the format of those texts by 
  clicking some buttons or menu items. In "markdown", user only need to use keyboard to type something to mark the
  texts. This approach can improve the productivity of documentation.
* "Markdown" is very popular. There are many editors are available in all platform. 
* "Markdown" can be translated into other format(such as, HTML) easily. That means, it is very easy to distribute 
 documents that are written in "markdown".
 

###Syntax Reference


###Syntax Exercise
[Markdown Tutorial](https://www.markdowntutorial.com/)


