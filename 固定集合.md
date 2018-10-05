## 概述

固定集合是固定大小的集合，支持基于插入顺序插入和检索文档的高吞吐量操作。固定集合以类似于循环缓冲区的方式工作：一旦集合占满其分配的空间，它通过覆盖集合中最旧的文档为新文档使用空间。









https://docs.mongodb.com/manual/core/capped-collections/