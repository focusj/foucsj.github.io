1. es中数据的组织形式：index对应传统数据库中database的概念，type对应传统数据库中table的概念。但是后续es将要删除type，因为这会误导使用者。在传统的数据库中，表是彼此隔离的，他们可以任意的设计自己的数据结构（名称类型映射）。但es中同名属性的类型映射必须是一致的，因为下层使用的是相同的lucene属性。
2. es提供了相对规范的REST API：http://localhost:9200/<index>/<type>/[<id>]。
