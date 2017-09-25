<h1><p style="text-align: center;">How To Write A Document</p></h1>



Overview
====================
This document is an reference about how to write a document in this repository. 
If you are new to this project and you are editing a document, please keep this document opening.
    
Table of content:
[TOC]

##Create An Document
The first step for creating an document is creating a folder for this document. The name of this folder
should reflect the subject of the document. The name of this folder should match regular expression
`^[a-z|A-Z|0-9|_]*$`. For example, the folder name for this document is "how_to_write_doc". This folder should
include all material for this document. Such as, markdown files, CSS files and images.

After the folder is created, an markdown document(suffix is ".md") with the same name should be created in this folder.
For example, a "how_to_write_doc.md" file is created for this document. This file is the entry point for the whole 
document. If there is too much content for a document, other "md" files can be created and they can be linked with 
hyperlink. However, this approach is not recommended because the "table of content" `[TOC]` will not be translated
properly.

##How To Use Markdown

###Why Markdown
