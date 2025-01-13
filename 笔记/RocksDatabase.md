## 1. 前言

是对RocksDB对象的整体封装

## 2. 内部类ColumnFamily

是对ColumnFamilyHandle的封装，ColumnFamilyHandle是ColumnFamily的文件描述符

ColumnFamily持有三个实例变量

* nameBytes 字节序列的列族名字
* name    string类型的列族名字
* ColumnFamilyHandle  被代理对象



