1. 把项目跑起来，调试跟进程序的运行
2. 明确自己的目的，带着问题来看代码
3. 分清主次，首先看主要结构，对于功能完整的模块，比如加密，编码则忽略的看
4. 纵向和横向，不断交互。依据模块，依据流程，彼此参照
5. 多写文档，多画流程图。文档建议这么写，先讲设计，再讲重要的数据结构·，比如leveldb中的memtable，sstable，iterator等，再讲关键流程比如compact
6. 梳理核心数据结构