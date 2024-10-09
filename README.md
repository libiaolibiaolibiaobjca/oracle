<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [GORM Oracle Driver](#gorm-oracle-driver)
  - [修改说明](#%E4%BF%AE%E6%94%B9%E8%AF%B4%E6%98%8E)
    - [修改内容](#%E4%BF%AE%E6%94%B9%E5%86%85%E5%AE%B9)
    - [为什么自定义](#%E4%B8%BA%E4%BB%80%E4%B9%88%E8%87%AA%E5%AE%9A%E4%B9%89)
  - [Description](#description)
  - [DB Driver](#db-driver)
  - [Required dependency Install](#required-dependency-install)
  - [Other](#other)
  - [Quick Start](#quick-start)
    - [how to install](#how-to-install)
    - [usage](#usage)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# GORM Oracle Driver

## 修改说明

### 修改内容

beefs 用到的列名称如果是关键字 "DELETE", "SIZE", "CIPHER", "LIMIT"，那么对其大写并添加双引号（"）

### 为什么自定义

开源的 gorm-oracle 都没办法正确读写数据

1. ["github.com/dzwvip/oracle"](https://github.com/dzwvip/oracle)

    ```text
        connectionUrl := go_ora.BuildUrl(hostname, port, database, user, password, nil)
        log.Printf("connecting to %s", connectionUrl)
        gormDB, err := gorm.Open(oracle.Open(connectionUrl), &gorm.Config{
            Logger: logger.Default.LogMode(logger.Info),
        })
    ```

    ```text
    2024-10-09 17:18:25.769 [INFO ] 21610 --- [26    ] [oracle2_store.go:75] : connecting to oracle://system:oracle@localhost:11521/XE
    
    2024/10/09 17:18:25 /Users/anan/Documents/DBJCA/beefs-1.7/weed/filer/oracle2/oracle2_store.go:76
    [error] failed to initialize database, got error ORA-00000: DPI-1047: Cannot locate a 64-bit Oracle Client library: "dlopen(libclntsh.dylib, 0x0001): tried: 'libclntsh.dylib' (no such file), '/System/Volumes/Preboot/Cryptexes/OSlibclntsh.dylib' (no such file), '/usr/lib/libclntsh.dylib' (no such file, not in dyld cache), 'libclntsh.dylib' (no such file), '/usr/local/lib/libclntsh.dylib' (no such file), '/usr/lib/libclntsh.dylib' (no such file, not in dyld cache)". See https://oracle.github.io/odpi/doc/installation.html#macos for help
    2024-10-09 17:18:25.776 [INFO ] 21610 --- [26    ] [configuration.go:25] : failed to initialize store for oracle2: can not connect to oracle://system:oracle@localhost:11521/XE error:ORA-00000: DPI-1047: Cannot locate a 64-bit Oracle Client library: "dlopen(libclntsh.dylib, 0x0001): tried: 'libclntsh.dylib' (no such file), '/System/Volumes/Preboot/Cryptexes/OSlibclntsh.dylib' (no such file), '/usr/lib/libclntsh.dylib' (no such file, not in dyld cache), 'libclntsh.dylib' (no such file), '/usr/local/lib/libclntsh.dylib' (no such file), '/usr/lib/libclntsh.dylib' (no such file, not in dyld cache)". See https://oracle.github.io/odpi/doc/installation.html#macos for help
    ```

2. oracle ["github.com/godoes/gorm-oracle"](https://github.com/godoes/gorm-oracle)

    ```shell
        options := map[string]string{
            "CONNECTION TIMEOUT": "90",
            "LANGUAGE":           "SIMPLIFIED CHINESE",
            "TERRITORY":          "CHINA",
            "SSL":                "false",
        }
        // oracle://user:password@127.0.0.1:1521/service
        connectionUrl := oracle.BuildUrl(hostname, port, database, user, password, options)
        dialector := oracle.New(oracle.Config{
            DSN:                     connectionUrl,
            IgnoreCase:              false, // query conditions are not case-sensitive
            NamingCaseSensitive:     false, // whether naming is case-sensitive
            VarcharSizeIsCharLength: true,  // whether VARCHAR type size is character length, defaulting to byte length
    
            // RowNumberAliasForOracle11 is the alias for ROW_NUMBER() in Oracle 11g, defaulting to ROW_NUM
            RowNumberAliasForOracle11: "ROW_NUM",
        })
        gormDB, err := gorm.Open(dialector, &gorm.Config{
            SkipDefaultTransaction:                   true, // 是否禁用默认在事务中执行单次创建、更新、删除操作
            DisableForeignKeyConstraintWhenMigrating: true, // 是否禁止在自动迁移或创建表时自动创建外键约束
            // 自定义命名策略
            NamingStrategy: schema.NamingStrategy{
                NoLowerCase:         false, // 是否不自动转换小写表名
                IdentifierMaxLength: 30,    // Oracle: 30, PostgreSQL:63, MySQL: 64, SQL Server、SQLite、DM: 128
            },
            PrepareStmt:     false, // 创建并缓存预编译语句，启用后可能会报 ORA-01002 错误
            CreateBatchSize: 50,    // 插入数据默认批处理大小
        })
    ```

   插入数据报错

    ```text
    [5.754ms] [rows:0] INSERT INTO BUCKET_META (NAME,BASIC_AUTH,CIPHER,SHARDS,CHUNK_SIZE,TTL,ALLOWED_TTLS,MAX_FILE_SIZE,LOCATION,DELETE,REVISE,LIMIT,EXTENDS,STORE,STATUS,OPT_STATUS,ROOT_KEY,VOLUME_ENCRYPT,KEEP_PLAIN,DAPS_BASE_URL,DAPS_APP_ID,DAPS_SECRET,REPLICATION,DISK_TYPE,COLLECTION,DATA_CENTER,RACK,FSYNC,VOLUME_GROWTH_COUNT,VERSION,CREATED_AT,UPDATED_AT) VALUES ('default','scott:tiger','','',4194304,'','',134217728,'{"type":"VolumeServer"}','{"allow":false}','{"status":"CLOSED","max":0}','{"allow":false,"files":0,"bytes":0,"deleteAllow":false,"folderDownloadAllow":false,"checkHash":false}','{"sdfConfig":{}}','','NORMAL','NORMAL','','',0,'','','','','','','','',0,0,0,'2024-10-09 16:57:22.155','2024-10-09 16:57:22.155')
    2024-10-09 16:57:22.161 [INFO ] 13534 --- [31    ] [bucket_meta.go:194] : txError: ORA-01747: user.table.column, table.column 或列说明无效
    
    2024/10/09 16:57:22 /Users/anan/Documents/DBJCA/beefs-1.7/weed/filer/abstract_sql/bucket_meta.go:208 ORA-01747: user.table.column, table.column 或列说明无效
    ```

    ```text
        options := map[string]string{
            "CONNECTION TIMEOUT": "90",
            "LANGUAGE":           "SIMPLIFIED CHINESE",
            "TERRITORY":          "CHINA",
            "SSL":                "false",
        }
        // oracle://user:password@127.0.0.1:1521/service
        connectionUrl := oracle.BuildUrl(hostname, port, database, user, password, options)
        dialector := oracle.New(oracle.Config{
            DSN:                     connectionUrl,
            IgnoreCase:              false, // query conditions are not case-sensitive
            NamingCaseSensitive:     true,  // whether naming is case-sensitive
            VarcharSizeIsCharLength: true,  // whether VARCHAR type size is character length, defaulting to byte length
    
            // RowNumberAliasForOracle11 is the alias for ROW_NUMBER() in Oracle 11g, defaulting to ROW_NUM
            RowNumberAliasForOracle11: "ROW_NUM",
        })
        gormDB, err := gorm.Open(dialector, &gorm.Config{
            SkipDefaultTransaction:                   true, // 是否禁用默认在事务中执行单次创建、更新、删除操作
            DisableForeignKeyConstraintWhenMigrating: true, // 是否禁止在自动迁移或创建表时自动创建外键约束
            // 自定义命名策略
            NamingStrategy: schema.NamingStrategy{
                NoLowerCase:         true, // 是否不自动转换小写表名
                IdentifierMaxLength: 30,   // Oracle: 30, PostgreSQL:63, MySQL: 64, SQL Server、SQLite、DM: 128
            },
            PrepareStmt:     false, // 创建并缓存预编译语句，启用后可能会报 ORA-01002 错误
            CreateBatchSize: 50,    // 插入数据默认批处理大小
        })
    ```

    ```text
    2024/10/09 17:03:38 /Users/anan/Documents/DBJCA/beefs-1.7/weed/filer/abstract_sql/bucket_meta.go:219 ORA-00942: 表或视图不存在
    
    [4.234ms] [rows:0] SELECT * FROM "BucketMeta" WHERE name = 'default' AND ROWNUM <= 1 ORDER BY "BucketMeta"."Name" 
    
    2024/10/09 17:03:38 /Users/anan/Documents/DBJCA/beefs-1.7/weed/filer/abstract_sql/bucket_meta.go:192 ORA-00942: 表或视图不存在
    
    [4.847ms] [rows:0] INSERT INTO "BucketMeta" ("Name","BasicAuth","Cipher","Shards","ChunkSize","TTL","AllowedTtls","MaxFileSize","Location","Delete","Revise","Limit","Extends","Store","Status","OptStatus","RootKey","VolumeEncrypt","KeepPlain","DapsBaseUrl","DapsAppID","DapsSecret","Replication","DiskType","Collection","DataCenter","Rack","Fsync","VolumeGrowthCount","Version","CreatedAt","UpdatedAt") VALUES ('default','scott:tiger','','',4194304,'','',134217728,'{"type":"VolumeServer"}','{"allow":false}','{"status":"CLOSED","max":0}','{"allow":false,"files":0,"bytes":0,"deleteAllow":false,"folderDownloadAllow":false,"checkHash":false}','{"sdfConfig":{}}','','NORMAL','NORMAL','','',0,'','','','','','','','',0,0,0,'2024-10-09 17:03:38.269','2024-10-09 17:03:38.269')
    2024-10-09 17:03:38.274 [INFO ] 15830 --- [56    ] [bucket_meta.go:194] : txError: ORA-00942: 表或视图不存在
    
    2024/10/09 17:03:38 /Users/anan/Documents/DBJCA/beefs-1.7/weed/filer/abstract_sql/bucket_meta.go:208 ORA-00942: 表或视图不存在
    
    ```

    ```text
    store.SqlGenerator = &abstract_sql.SqlGen{
            CreateTableSqlTemplate:              createTable,
            CreateTableBucketMetaSqlTemplate:    createTableBucketMeta,
            CreateTableStoreMetaSqlTemplate:     createTableStoreMeta,
            CreateTableBucketStatsSqlTemplate:   createTableBucketStats,
            CreateTableWorkOrderMetaSqlTemplate: createTableWorkOrderMeta,
            DropTableSqlTemplate:                `drop table "%s"`,
            TruncateTableSqlTemplate:            `truncate table "%s"`,
            UpsertQueryTemplate:                 upsertQuery,
            Dialect:                             abstract_sql.OracleWrapper,
        }
    
        options := map[string]string{
            "CONNECTION TIMEOUT": "90",
            "LANGUAGE":           "SIMPLIFIED CHINESE",
            "TERRITORY":          "CHINA",
            "SSL":                "false",
        }
        // oracle://user:password@127.0.0.1:1521/service
        connectionUrl := oracle.BuildUrl(hostname, port, database, user, password, options)
        dialector := oracle.New(oracle.Config{
            DSN:                     connectionUrl,
            IgnoreCase:              false, // query conditions are not case-sensitive
            NamingCaseSensitive:     true,  // whether naming is case-sensitive
            VarcharSizeIsCharLength: true,  // whether VARCHAR type size is character length, defaulting to byte length
    
            // RowNumberAliasForOracle11 is the alias for ROW_NUMBER() in Oracle 11g, defaulting to ROW_NUM
            RowNumberAliasForOracle11: "ROW_NUM",
        })
        gormDB, err := gorm.Open(dialector, &gorm.Config{
            SkipDefaultTransaction:                   true, // 是否禁用默认在事务中执行单次创建、更新、删除操作
            DisableForeignKeyConstraintWhenMigrating: true, // 是否禁止在自动迁移或创建表时自动创建外键约束
            // 自定义命名策略
            NamingStrategy: schema.NamingStrategy{
                NoLowerCase:         false, // 是否不自动转换小写表名
                IdentifierMaxLength: 30,    // Oracle: 30, PostgreSQL:63, MySQL: 64, SQL Server、SQLite、DM: 128
            },
            PrepareStmt:     false, // 创建并缓存预编译语句，启用后可能会报 ORA-01002 错误
            CreateBatchSize: 50,    // 插入数据默认批处理大小
        })
        
    ```

    ```text
    2024/10/09 17:10:44 /Users/anan/Documents/DBJCA/beefs-1.7/weed/filer/abstract_sql/bucket_meta.go:192 ORA-00942: 表或视图不存在
    
    [3.380ms] [rows:0] INSERT INTO "bucket_meta" ("name","basic_auth","cipher","shards","chunk_size","ttl","allowed_ttls","max_file_size","location","delete","revise","limit","extends","store","status","opt_status","root_key","volume_encrypt","keep_plain","daps_base_url","daps_app_id","daps_secret","replication","disk_type","collection","data_center","rack","fsync","volume_growth_count","version","created_at","updated_at") VALUES ('default','scott:tiger','','',4194304,'','',134217728,'{"type":"VolumeServer"}','{"allow":false}','{"status":"CLOSED","max":0}','{"allow":false,"files":0,"bytes":0,"deleteAllow":false,"folderDownloadAllow":false,"checkHash":false}','{"sdfConfig":{}}','','NORMAL','NORMAL','','',0,'','','','','','','','',0,0,0,'2024-10-09 17:10:44.299','2024-10-09 17:10:44.299')
    2024-10-09 17:10:44.303 [INFO ] 18441 --- [69    ] [bucket_meta.go:194] : txError: ORA-00942: 表或视图不存在
    
    2024/10/09 17:10:44 /Users/anan/Documents/DBJCA/beefs-1.7/weed/filer/abstract_sql/bucket_meta.go:208 ORA-00942: 表或视图不存在
    
    ```

## Description

GORM Oracle driver for connect Oracle DB and Manage Oracle DB, Based
on [CengSin/oracle](https://github.com/CengSin/oracle)
，not recommended for use in a production environment

## DB Driver

[godror](https://github.com/godror/godror)

## Required dependency Install

- Oracle 12C+
- Golang 1.13+
- see [ODPI-C Installation.](https://oracle.github.io/odpi/doc/installation.html)
- gorm 1.24.0+
- godror 0.33+

## Other

Another library that uses the go-ora driver, [gorm-oracle](https://github.com/dzwvip/gorm-oracle), does not require the
installation of an Oracle client

## Quick Start

### how to install

```bash
go get github.com/dzwvip/oracle
```

### usage

```go
import (
"fmt"
"github.com/dzwvip/oracle"
"gorm.io/gorm"
"log"
)

func main() {
dsn := "oracle://system:password@127.0.0.1:1521/orcl"
db, err := gorm.Open(oracle.Open(dsn), &gorm.Config{})
if err != nil {
// panic error or log error info
} 

// do somethings
}
```
