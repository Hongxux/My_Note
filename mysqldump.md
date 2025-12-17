
`mysqldump`是 MySQL 官方提供的命令行工具，用于对数据库进行**逻辑备份**。其产生的备份文件是包含 SQL 语句的文本文件，可用于重建数据库。

#### 一、 工具定位与输出内容

- ​**核心用途**​：用于**备份数据库**或在不同 MySQL 数据库之间进行**数据迁移**。
    
- ​**备份内容**​：备份文件包含一系列 SQL 语句，主要是**创建表结构的 `CREATE TABLE`语句**和**插入数据的 `INSERT INTO`语句**。当执行这个备份文件时，可以原样重现备份时的数据。
    

#### 二、 三种备份语法模式（对应不同备份范围）

图中列出了三种基本命令形式，决定了备份的广度：

1. ​**备份单个数据库（可指定具体表）​**​
    
    ```
    mysqldump [options] db_name [tables]
    ```
    
    - ​**示例**​：`mysqldump -u root -p mydb users orders`仅备份 `mydb`数据库中的 `users`和 `orders`表。
        
    - 不指定表名则备份整个数据库。
        
    
2. ​**备份指定的多个数据库**​
    
    ```
    mysqldump [options] --databases db1 [db2 db3...]
    # 或简写
    mysqldump [options] -B db1 [db2 db3...]
    ```
    
    - ​**示例**​：`mysqldump -u root -p --databases db1 db2`同时备份 `db1`和 `db2`两个数据库。
        
    - 与模式1的关键区别：备份文件中会包含 `CREATE DATABASE IF NOT EXISTS`语句，便于在恢复时自动创建数据库。
        
    
3. ​**备份所有数据库**​
    
    ```
    mysqldump [options] --all-databases
    # 或简写
    mysqldump [options] -A
    ```
    
    - ​**示例**​：`mysqldump -u root -p --all-databases > full_backup.sql`备份服务器上所有数据库。
        
    - ​**用途**​：通常用于对整个 MySQL 实例进行完整备份。
        
    

#### 三、 关键选项详解

1. ​**连接选项**​：用于指定如何连接到你想要备份的数据库服务器。
    
    - `-u`：指定用户名。
        
    - `-p`：指定密码（建议只在 `-p`后不跟密码，回车后单独输入，以保证安全）。
        
    - `-h`：指定数据库服务器 IP 或主机名，默认为 `localhost`。
        
    - `-P`：指定连接端口，默认为 `3306`。
        
    
2. ​**输出选项**​：控制备份文件内容的生成规则，这是最灵活和重要的部分。
    
    - `--add-drop-database`/ `--add-drop-table`：
        
        - 在 `CREATE DATABASE`/ `CREATE TABLE`语句前加上 `DROP DATABASE IF EXISTS`/ `DROP TABLE IF EXISTS`语句。
            
        - ​**作用**​：确保恢复时是纯净的环境，避免因表已存在而报错。`--add-drop-table`默认开启。
            
        
    - `-n`(`--no-create-db`)：
        
        - 备份文件中**不包含**​ `CREATE DATABASE`语句。
            
        - ​**适用场景**​：在模式2或模式3备份时，如果你不想自动创建数据库，可以使用此选项。
            
        
    - `-t`(`--no-create-info`)：
        
        - 备份文件中**不包含**创建表结构的 `CREATE TABLE`语句。
            
        - ​**备份结果**​：只有纯数据（INSERT 语句）。
            
        - ​**适用场景**​：只需备份数据，表结构已存在或由其他方式创建。
            
        
    - `-d`(`--no-data`)：
        
        - 备份文件中**不包含**数据（即没有 INSERT 语句）。
            
        - ​**备份结果**​：只有创建表结构的 SQL。
            
        - ​**适用场景**​：只需备份表结构（Schema），用于部署或分析。
            
        
    - `-T`(`--tab`)：​**一个非常实用的选项**。
        
        - 为每个表**自动生成两个文件**。
            
        - 一个 `.sql`文件：仅包含创建该表结构的语句。
            
        - 一个 `.txt`文件（默认定界符格式）：包含该表的所有数据。
            
        - ​**优势**​：数据文本文件可以被其他程序（如 Excel、Python pandas）轻松读取，便于数据交换和分析。
            
        - ​**注意**​：使用此选项时，需要指定文件输出目录，且 MySQL 用户需要有该目录的写权限。
            
        
    

### 总结与常用命令示例

|场景|示例命令|说明|
|---|---|---|
|​**完整备份单个数据库**​|`mysqldump -u root -p --databases myblog > myblog_backup.sql`|备份 `myblog`库，包含建库语句。|
|​**仅备份表结构**​|`mysqldump -u root -p -d myblog > myblog_schema_only.sql`|仅备份 `myblog`库的所有表结构。|
|​**仅备份数据**​|`mysqldump -u root -p -t myblog > myblog_data_only.sql`|仅备份 `myblog`库的所有数据。|
|​**备份全库**​|`mysqldump -u root -p --all-databases > full_backup.sql`|备份整个 MySQL 实例。|
|​**生成结构文件和数据文件**​|`mysqldump -u root -p -T /tmp/export/ myblog`|在 `/tmp/export/`目录下为 `myblog`库的每个表生成 `.sql`（结构）和 `.txt`（数据）文件。|

​**核心要点**​：`mysqldump`的核心优势在于其灵活性和通用性（SQL 标准格式）。理解并熟练运用这些**输出选项**，你可以轻松定制出满足各种特定需求（如只要结构、只要数据、结构数据分离）的备份方案，是 DBA 和开发者的必备技能。



### 两种数据导入方法**`mysqlimport`**​ 和 ​**`source`命令

这张图的核心是教你如何将备份的数据文件重新导入（恢复）到 MySQL 数据库中。根据备份时使用的工具和生成的文件格式，需要选择对应的导入方法。

为了更直观地理解这两种方法的分工与流程，下图清晰地展示了它们各自的适用场景和在数据流转中的位置：

```
flowchart TD
A[数据备份源] --> B{备份方式与文件格式}
B --> C[“mysqldump -T<br>（生成文本数据文件）”]
B --> D[“mysqldump<br>（生成标准SQL文件）”]

C --> E[“导入工具: mysqlimport<br>（专用于格式化文本文件）”]
D --> F[“导入工具: source 命令<br>（专用于执行SQL脚本）”]

E --> G[成功恢复数据至数据库]
F --> G
```

#### 一、 `mysqlimport`工具：用于特定格式的文本文件

​**1. 定位与用途**​

- `mysqlimport`是一个**命令行客户端工具**。
    
- 它专门用于导入由 ​**`mysqldump -T`**​ 选项导出的那种**纯文本格式的数据文件**​（如 `.txt`文件）。
    

​**2. 工作原理**​

- 它实际上是 `LOAD DATA INFILE`SQL 语句的一个**命令行封装**，提供了从命令行直接加载数据文件的便捷接口。
    
- 它直接与 MySQL 服务器交互，将数据文件的内容加载到指定的表中。
    

​**3. 语法与示例**​

```
mysqlimport [options] db_name textfile1 [textfile2...]
```

- ​**`db_name`**​：指定要导入数据的**数据库名**。
    
- ​**`textfile1`**​：数据文件的**完整路径**。​**关键点：​**​ 这个文件名（不包括路径和扩展名）必须与目标**表名**一致。
    
    - 例如：要导入到 `city`表，文件必须命名为 `city.txt`。
        
    

​**示例解读：​**​

```
mysqlimport -uroot -p2143 test /tmp/city.txt
```

- ​**`-uroot -p2143`**​：使用 root 用户和密码连接数据库。
    
- ​**`test`**​：数据将导入到 `test`数据库中。
    
- ​**`/tmp/city.txt`**​：从该路径读取数据文件。`mysqlimport`会认为目标表名是 `city`（根据文件名 `city.txt`推断）。
    

​**4. 常用选项**​

- `--fields-terminated-by=`：指定字段之间的分隔符（如 `--fields-terminated-by=','`）。
    
- `--lines-terminated-by=`：指定行之间的分隔符（如 `--lines-terminated-by='\n'`）。
    
- `--ignore`或 `--replace`：处理唯一键冲突的方式（忽略或替换重复记录）。
    

#### 二、 `source`命令：用于执行 SQL 脚本文件

​**1. 定位与用途**​

- `source`（或其缩写 `\.`）是一个在 ​**MySQL 命令行客户端内部**执行的命令。
    
- 它用于读取并执行一个包含 ​**SQL 语句**的文件（通常是 `.sql`后缀）。这种文件通常由标准的 `mysqldump`（不加 `-T`参数）命令生成。
    

​**2. 工作原理**​

- 它在 MySQL 客户端环境中，将指定文件中的 SQL 语句（如 `CREATE TABLE`, `INSERT INTO`）​**逐条执行**，就像你手动输入这些命令一样。
    

​**3. 语法与示例**​

```
-- 首先，需要登录到 MySQL 客户端
mysql -uroot -p

-- 然后，在 mysql> 提示符下使用 source 命令
mysql> source /root/backup.sql;
-- 或者使用缩写
mysql> \. /root/backup.sql
```

​**示例解读：​**​

```
source /root/xxxx.sql;
```

- 这个命令会在当前连接的数据库中，执行 `/root/xxxx.sql`文件中的所有 SQL 语句。
    
- 如果 `.sql`文件是由 `mysqldump --databases`导出的，它通常会包含创建数据库和选择数据库的语句，可以完整恢复整个数据库结构和数据。
    

### 总结与选择建议

|特性|`mysqlimport`|`source`命令|
|---|---|---|
|​**运行环境**​|​**操作系统命令行**​|​**MySQL 客户端内部**​（`mysql>`提示符下）|
|​**导入文件类型**​|​**纯文本数据文件**​（如 `.txt`），通常由 `mysqldump -T`产生|​**SQL 脚本文件**​（如 `.sql`），由标准 `mysqldump`产生|
|​**功能**​|快速将格式化文本数据加载到**已存在的表**中|执行 SQL 脚本，可包含**建库、建表、插入数据**等所有操作|
|​**适用场景**​|批量导入大量数据，数据交换|完整的数据库备份恢复，数据库迁移|

​**简单来说：​**​

- 如果你的备份文件是 ​**`.sql`文件**​（里面是 SQL 语句），请使用 ​**`source`**​ 命令在 MySQL 客户端内执行。
    
- 如果你的备份文件是 ​**`.txt`文本数据文件**​（只有数据，没有表结构），并且表已经存在，请使用 ​**`mysqlimport`**​ 工具从操作系统命令行导入。
    

掌握这两种方法，你就能应对绝大多数 MySQL 数据恢复和迁移的需求。