---
title: SourceKit -- source.request.docinfo
---
# {{ page.title }}

*Valid July 2017 with swift:master*

Quest: find out how this works.

## Flow

* Requests.cpp:handleSemanticRequest()
    * Requests.cpp:reportDocInfo()
        * Create SKDocConsumer
        * SwiftDocSupport.cpp:SwiftLangSupport::getDocInfo()
            * SwiftDocSupport.cpp:reportSourceDocInfo()
                * Create compiler and compile up to sema
                * Collect Decls from AST into SourceTextInfo
                * reportDocEntities() to push structure data back to consumer
                    * initDocEntityInfo()
                    * reportRelated()
                    * reportAttributes()
                * reportSourceAnnotations() to push syntax data back to consumer

## initDocEntityInfo

*notes on interesting bits of extracting from AST*

## SKDocConsumer

So then these fields are returned in the docinfo element:
