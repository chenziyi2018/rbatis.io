* [中文](https://rbatis.github.io/rbatis.io/#/)

### Support database

| database    | support |
| ------ | ------ |
| Mysql            | √     |   
| Postgres         | √     |  
| Sqlite           | √     |  
| Mssql/Sqlserver           | √     |  
| MariaDB(Mysql)             | √     |
| TiDB(Mysql)             | √     |
| CockroachDB(Postgres)      | √     |

# Rbatis-init

> Install dependencies

``` toml
#json support(must)
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

#date(must)
chrono = { version = "0.4", features = ["serde"] }

#log support(must)
log = "0.4"
fast_log="1.3"

#BigDecimal support(not must)
bigdecimal = "0.2"

#rbatis support(must)
rbatis =  { version = "1.8" }
```

> Ordinary init

```rust
#[macro_use]
extern crate rbatis;

let rb = Rbatis::new();
///Connect to the database, automatic judgment drive type "mysql: / / *", "postgres: / / *", "sqlite: / / *","mssql://*"  load driver  
rb.link("mysql://root:123456@localhost:3306/test").await.unwrap();
///Customize connection pool parameters. (optional)
// use crate::core::db::DBPoolOptions;
// let mut opt =DBPoolOptions::new();
// opt.max_connections=100;
// rb.link_opt("mysql://root:123456@localhost:3306/test",&opt).await.unwrap();

//With log output enabled, you can also use other logging frameworks, which are not qualified
fast_log::init_log("requests.log", 1000,log::Level::Info,true);
```

> Initialize with a global variable (depending on the library lazy_static)

```rust
#[macro_use]
extern crate rbatis;
use rbatis::crud::CRUD;

lazy_static! {
  // Rbatis is thread-safe, and the runtime method is Send+Sync. there is no need to worry about thread contention
  static ref RB:Rbatis=Rbatis::new();
}
//Use the async_STd main method here, you can select actix, ToKIO, and other runtime main methods or spawn
#[async_std::main]
async fn main() {
      fast_log::init_log("requests.log", 1000,log::Level::Info,true);
      RB.link("mysql://root:123456@localhost:3306/test").await.unwrap();
}

```

# CRUDTable

> CRUDTable An interface is a Trait that helps define the table structure, and it provides the following methods

* IdType (The id field type corresponding to the struct must be declared)
* id_name() The name of the primary key ID (non-mandatory, default ID)
* table_name() Table name (serpentine name corresponding to struct, optional rewrite)
* table_columns() Comma-separated string of table fields (all field names corresponding to the struct, optionally
  overridden)
* format_chain() Field formatting chain (you can format fields such as Pg database string date to timestamp #{date}::
  Timestamp, optional override)

  
> use the Attr attribute macro to achieve CRUDTable, which is more scalable and allows you to customize table names and fields

| attr    | doc |
| ------ | ------ |
| id_name | the table id name |
| id_type | the table id type |
| table_name | table name |
| table_columns | table columns use ',' split |
| formats_pg,formats_postgres | Postgres Column SQL formatting for type conversion|
| formats_mysql | Mysql Column SQL formatting for type conversion|
| formats_sqlite | Sqlite Column SQL formatting for type conversion|
| formats_mssql | Mssql Column SQL formatting for type conversion|

```rust
//for example-1(All automatic generation):

    #[crud_enable]
    #[derive(Clone, Debug)]
    pub struct BizActivity {
        pub id: Option<String>,
        pub name: Option<String>,
        pub delete_flag: Option<i32>,
    }
// for example-2（Only the table name is changed, the rest is generated automatically）:
    #[crud_enable(table_name:biz_activity)]
    #[derive(Clone, Debug)]
    pub struct BizActivity {
        pub id: Option<String>,
        pub name: Option<String>,
        pub delete_flag: Option<i32>,
    }
//for example-3（Full customization）:
    #[crud_enable( id_name:"id" |  id_type:"String" | table_name:"biz_activity" | table_columns:"id,name,delete_flag" | formats_pg:"id:{}::uuid")]
    #[derive(Clone, Debug)]
    pub struct BizActivity {
        pub id: Option<String>,
        pub name: Option<String>,
        pub delete_flag: Option<i32>,
    }
    //for example-4（Full customization）:
    #[crud_enable( id_name:"id" |  id_type:"String" | table_name:"biz_activity" | table_columns:"id,name,delete_flag" | formats_pg:"id:{}::uuid")]
    #[derive(Clone, Debug)]
    pub struct BizActivity {
        pub id: Option<String>,
        pub name: Option<String>,
        pub delete_flag: Option<i32>,
    }
```

> (Optional) derive macro impl CRUDTable  is that the macros generate code in the compiler

```rust
#[macro_use]
extern crate rbatis;

#[derive(CRUDTable,Serialize, Deserialize, Clone, Debug)] 
pub struct BizActivity {    //will be table_name BizActivity => "biz_activity"
    pub id: Option<String>, 
    pub name: Option<String>,
    pub pc_link: Option<String>,
    pub h5_link: Option<String>,
    pub pc_banner_img: Option<String>,
    pub h5_banner_img: Option<String>,
    pub sort: Option<String>,
    pub status: Option<i32>,
    pub remark: Option<String>,
    pub create_time: Option<NaiveDateTime>,
    pub version: Option<i32>,
    pub delete_flag: Option<i32>,
}
```

> (Optional) Or using IMPL to achieve CRUDTable has the benefit of high custom controllability and can reduce JSON serialization if you override methods such as field_name

```rust
    use rbatis::crud::CRUDTable;
    impl CRUDTable for BizActivity {
        type IdType = String; //By default, IdType is provided; other methods in the interface use JSON serialization by default
        fn get_id(&self) -> Option<&Self::IdType>; // must be add impl this
        //fn table_name() -> String {} //Can be rewritten
        //fn table_columns() -> String {}  //Can be rewritten
        //fn format_chain() -> Vec<Box<dyn ColumnFormat>>{} //Can be rewritten
    }
```

## database column formatting macro

> a Postgres database uses UUID as the primary key and fails to precompile if a string parameter is passed in under precompiled SQL.
> therefore requires a Pg database '::type' to cast, using column formatting macros The
> format macro is defined as formats_database:"column_name:format_string"  format_string with a {} symbol
> format macro use ',' split columns
> for example:

```rust
#[crud_enable(formats_pg:"id:{}::uuid")]
//#[crud_enable(formats_pg:"id:{}::uuid,create_time:{}::timestamp")]
//#[crud_enable(formats_mysql:...)]
//#[crud_enable(formats_sqlite:...)]
//#[crud_enable(formats_mssql:...)]
//#[crud_enable(formats_mssql:...|formats_pg:...)|...]
```

```rust
//this is example format.
#[crud_enable(formats_pg:"id:{}::uuid")]
#[derive(Clone, Debug)]
pub struct BizUuid {
    pub id: Option<Uuid>,
    pub name: Option<String>,
}
#[async_std::test]
    pub async fn test_postgres_uuid() {
        fast_log::init_log("requests.log", 1000, log::Level::Info, None, true);
        let rb = Rbatis::new();
        rb.link("postgres://postgres:123456@localhost:5432/postgres").await.unwrap();
        let uuid = Uuid::from_str("df07fea2-b819-4e05-b86d-dfc15a5f52a9").unwrap();
        //create table
        rb.exec("", "CREATE TABLE biz_uuid( id uuid, name VARCHAR, PRIMARY KEY(id));").await;
        //insert table
        rb.save("", &BizUuid { id: Some(uuid), name: Some("test".to_string()) }).await;
        //update table
        rb.update_by_id("",&BizUuid{ id: Some(uuid.clone()), name: Some("test_updated".to_string()) }).await;
        //query table
        let data: BizUuid = rb.fetch_by_id("", &uuid).await.unwrap();
        println!("{:?}", data);
        //delete table
        rb.remove_by_id::<BizUuid>("",&uuid).await;
    }
2020-12-14 14:26:58.072638 +08:00    INFO rbatis::plugin::log - [rbatis] [] Exec  ==> CREATE TABLE biz_uuid( id uuid, name VARCHAR, PRIMARY KEY(id));
2020-12-14 14:26:58.084423 +08:00    INFO rbatis::plugin::log - [rbatis] [] Exec  ==> INSERT INTO biz_uuid (id,name) VALUES ($1::uuid,$2)
                                                                [rbatis] [] Args  ==> ["df07fea2-b819-4e05-b86d-dfc15a5f52a9","test"]
2020-12-14 14:26:58.093312 +08:00    INFO rbatis::plugin::log - [rbatis] [] Exec  ==> UPDATE biz_uuid SET  name = $1 WHERE id = $2::uuid
                                                                [rbatis] [] Args  ==> ["test_updated","df07fea2-b819-4e05-b86d-dfc15a5f52a9"]
2020-12-14 14:26:58.103995 +08:00    INFO rbatis::plugin::log - [rbatis] [] Query ==> SELECT id,name FROM biz_uuid WHERE id = $1::uuid
                                                                [rbatis] [] Args  ==> ["df07fea2-b819-4e05-b86d-dfc15a5f52a9"]
2020-12-14 14:26:58.125965 +08:00    INFO rbatis::plugin::log - [rbatis] [] Exec  ==> DELETE FROM biz_uuid WHERE id = $1::uuid
                                                                [rbatis] [] Args  ==> ["df07fea2-b819-4e05-b86d-dfc15a5f52a9"]
```

# Wrapper

| method    | sql |
| ------ | ------ |
| and            |  AND     |   
| or         | OR     |  
| having           | HAVING {}     |  
| all_eq(map[String,#{}#{}])             | a = #{}, b= #{},c=#{}     |
| eq      |  a = #{}    |
| ne      |  a <> #{}    |
| order_by(bool,&[str])      |  order by #{} desc, #{} asc....    |
| group_by      |  group by #{},#{},#{}    |
| gt      |  a > #{}    |
| ge      |  a >= #{}    |
| lt      |  a < #{}    |
| le      |  a <= #{}    |
| between(column,min,max)      |  BETWEEN #{} AND #{}    |
| not_between(column,min,max)      |  NOT BETWEEN #{} AND #{}    |
| like(column,obj)      |   LIKE '%#{}%'   |
| like_left(column,obj)      |   LIKE '%#{}'   |
| like_right(column,obj)      |   LIKE '#{}%'   |
| not_like(column,obj)      |   NOT LIKE  '%#{}%'   |
| is_null(column)      |   #{} IS NULL   |
| is_not_null(column)      |   #{} IS NOT NULL   |
| r#in(column,args)      |   IN (#{},#{},#{},#{})  |
| not_in(column,args)      |   NOT IN (#{},#{},#{},#{})  |

> Wrapper Special methods

| method    | sql/rust_code | features |
| ------ | ------ |------ |
| push_wrapper(sql,wrapper)            |  'SELECT * FROM TABLE'=> 'SELECT * FROM TABLE #{sql}'     |  Wrapper add wrapper    |   
| push(sql,args)            |  'SELECT * FROM TABLE'=> 'SELECT * FROM TABLE #{sql}'     |   SQL and parameters are added to the Wrapper   |   
| push_sql(sql)            |  'SELECT * FROM TABLE'=> 'SELECT * FROM TABLE #{sql}'     |    Add SQL wrapper  |   
| push_arg(arg)            |   wrapper.push(*)     |   Wrapper add parameter   |   
| do_if(test:bool,method)            |  wrapper.do_if(p.is_some(), *)    |  Wrapper execution judgment    |   
| do_match(&[method...])  |  wrapper.do_match(p.is_some(), *))    |    The Wrapper performs a match condition match  |   

> Wrapper use case

```rust
  //should use for example:    let RB=Rbatis::new();   RB.new_wrapper() or RB.new_wrapper_table()
  let w = Wrapper::new(&DriverType::Mysql).eq("id", 1)
            .ne("id", 1)
            .r#in("id", &[1, 2, 3])
            .not_in("id", &[1, 2, 3])
            .all_eq(&m)
            .like("name", 1)
            .or()
            .not_like("name", "asdf")
            .between("create_time", "2020-01-01 00:00:00", "2020-12-12 00:00:00")
            .group_by(&["id"])
            .order_by(true, &["id", "name"])
            .;
```

# CRUD

```rust
let rb = Rbatis::new();
rb.link("mysql://root:123456@localhost:3306/test").await.unwrap();

let activity = BizActivity {
                id: Some("12312".to_string()),
                name: None,
                remark: None,
                create_time: Some(NaiveDateTime::now()),
                version: Some(1),
                delete_flag: Some(1),
            };
///save
rb.save("",&activity).await;
//Exec ==> INSERT INTO biz_activity (create_time,delete_flag,h5_banner_img,h5_link,id,name,pc_banner_img,pc_link,remark,sort,status,version) VALUES ( ? , ? , ? , ? , ? , ? , ? , ? , ? , ? , ? , ? )

///save batch
rb.save_batch("", &vec![activity]).await;
//Exec ==> INSERT INTO biz_activity (create_time,delete_flag,h5_banner_img,h5_link,id,name,pc_banner_img,pc_link,remark,sort,status,version) VALUES ( ? , ? , ? , ? , ? , ? , ? , ? , ? , ? , ? , ? ),( ? , ? , ? , ? , ? , ? , ? , ? , ? , ? , ? , ? )

///The query, Option wrapper, is None if the data is not found
let result: Option<BizActivity> = rb.fetch_by_id("", &"1".to_string()).await.unwrap();
//Query ==> SELECT create_time,delete_flag,h5_banner_img,h5_link,id,name,pc_banner_img,pc_link,remark,sort,status,version  FROM biz_activity WHERE delete_flag = 1  AND id =  ? 

///query all
let result: Vec<BizActivity> = rb.fetch_list("").await.unwrap();
//Query ==> SELECT create_time,delete_flag,h5_banner_img,h5_link,id,name,pc_banner_img,pc_link,remark,sort,status,version  FROM biz_activity WHERE delete_flag = 1

///query id vec, return vec result
let result: Vec<BizActivity> = rb.fetch_list_by_ids("",&["1".to_string()]).await.unwrap();
//Query ==> SELECT create_time,delete_flag,h5_banner_img,h5_link,id,name,pc_banner_img,pc_link,remark,sort,status,version  FROM biz_activity WHERE delete_flag = 1  AND id IN  (?) 

///custom  query(use Wrapper)
let w = rb.new_wrapper().eq("id", "1").;
let r: Result<Option<BizActivity>, Error> = rb.fetch_by_wrapper("", &w).await;
//Query ==> SELECT  create_time,delete_flag,h5_banner_img,h5_link,id,name,pc_banner_img,pc_link,remark,sort,status,version  FROM biz_activity WHERE delete_flag = 1  AND id =  ? 

///delete
rb.remove_by_id::<BizActivity>("", &"1".to_string()).await;
//Exec ==> UPDATE biz_activity SET delete_flag = 0 WHERE id = 1

///delete batch
rb.remove_batch_by_id::<BizActivity>("", &["1".to_string(), "2".to_string()]).await;
//Exec ==> UPDATE biz_activity SET delete_flag = 0 WHERE id IN (  ?  ,  ?  ) 

///update(use Wrapper)
let w = rb.new_wrapper().eq("id", "12312").;
rb.update_by_wrapper("", &activity, &w).await;
//Exec ==> UPDATE biz_activity SET  create_time =  ? , delete_flag =  ? , status =  ? , version =  ?  WHERE id =  ? 
}

///...For more, check out CRUd.rs
```

# Expr-OperationalExpression

> expressions are used for operations on parameters, such as String CONCAT,addition, subtraction, multiplication, and division, square, mod, parameter (A.B.C), array (a[0]), comparison, etc...
> expressions are commonly found in py_sql if conditions, such as #{} or ${} expressions
> The operators supported by the > operand expression engine are shown below

| token    | doc  |
| ------ | ------ |
|   ()    |    brackets    | 
|   %     |        | 
|   ^     |   xor     | 
|   *     |        | 
|   **     |   square     | 
|   /     |        | 
|   +     |        | 
|   -     |        | 
|   <=     |        | 
|   <     |        | 
|   >     |        | 
|   > =     |        | 
|   !=     |        | 
|   ==     |        | 
|   &&     |        | 
|&#124;&#124;|        | 

> operating-expression syntax example

``` rust
    #[test]
    fn test_node_run() {
        let runtime = RExprRuntime::new();
        let arg = json!({"a":1,"b":2,"c":"c", "d":null,});
        let exec_expr = |arg: &serde_json::Value, expr: &str| -> serde_json::Value{
            println!("{}", expr);
            runtime.eval(expr, arg).unwrap()
        };
        assert_eq!(exec_expr(&arg, "-1 == -a"), json!(true));
        assert_eq!(exec_expr(&arg, "d.a == null"), json!(true));
        assert_eq!(exec_expr(&arg, "1 == 1.0"), json!(true));
        assert_eq!(exec_expr(&arg, "'2019-02-26' == '2019-02-26'"), json!(true));
        assert_eq!(exec_expr(&arg, "`f`+`s`"), json!("fs"));
        assert_eq!(exec_expr(&arg, "a +1 > b * 8"), json!(false));
        assert_eq!(exec_expr(&arg, "a >= 0"), json!(true));
        assert_eq!(exec_expr(&arg, "'a'+c"), json!("ac"));
        assert_eq!(exec_expr(&arg, "b"), json!(2));
        assert_eq!(exec_expr(&arg, "a < 1"), json!(false));
        assert_eq!(exec_expr(&arg, "a +1 > b*8"), json!(false));
        assert_eq!(exec_expr(&arg, "a * b == 2"), json!(true));
        assert_eq!(exec_expr(&arg, "a - b == 0"), json!(false));
        assert_eq!(exec_expr(&arg, "a >= 0 && a != 0"), json!(true));
        assert_eq!(exec_expr(&arg, "a == 1 && a != 0"), json!(true));
        assert_eq!(exec_expr(&arg, "1 > 3 "), json!(false));
        assert_eq!(exec_expr(&arg, "1 + 2 != null"), json!(true));
        assert_eq!(exec_expr(&arg, "1 != null"), json!(true));
        assert_eq!(exec_expr(&arg, "1 + 2 != null && 1 > 0 "), json!(true));
        assert_eq!(exec_expr(&arg, "1 + 2 != null && 2 < b*8 "), json!(true));
        assert_eq!(exec_expr(&arg, "-1 != null"), json!(true));
        assert_eq!(exec_expr(&arg, "-1 != -2 && -1 == 2-3 "), json!(true));
        assert_eq!(exec_expr(&arg, "-3 == b*-1-1 "), json!(true));
        assert_eq!(exec_expr(&arg, "0-1 + a*0-1 "), json!(-2));
        assert_eq!(exec_expr(&arg, "2 ** 3"), json!(8.0));
        assert_eq!(exec_expr(&arg, "0-1 + -1*0-1 "), json!(-2));
        assert_eq!(exec_expr(&arg, "1-"), json!(1));
        assert_eq!(exec_expr(&arg, "-1"), json!(-1));
        assert_eq!(exec_expr(&arg, "1- -1"), json!(1--1));
        assert_eq!(exec_expr(&arg, "1-2 -1+"), json!(1-2-1));
        assert_eq!(exec_expr(&arg, "e[1]"), json!(null));
        assert_eq!(exec_expr(&arg, "e[0]"), json!(1));
        assert_eq!(exec_expr(&arg, "f[0].field"), json!(1));
        assert_eq!(exec_expr(&arg, "f.0.field"), json!(1));
        assert_eq!(exec_expr(&arg, "0.1"), json!(0.1));
        assert_eq!(exec_expr(&arg, "1"), json!(1));
        assert_eq!(exec_expr(&arg, "(1+1)"), json!(2));
        assert_eq!(exec_expr(&arg, "(1+5)>5"), json!((1+5)>5));
        assert_eq!(exec_expr(&arg, "(18*19)<19*19"), json!((18*19)<19*19));
        assert_eq!(exec_expr(&arg, "2*(1+1)"), json!(2*(1+1)));
        assert_eq!(exec_expr(&arg, "2*(1+(1+1)+1)"), json!(2*(1+(1+1)+1)));
        assert_eq!(exec_expr(&arg, "(((34 + 21) / 5) - 12) * 348"), json!((((34 + 21) / 5) - 12) * 348));
        assert_eq!(exec_expr(&arg, "null ^ null"), json!(0 ^ 0));
        assert_eq!(exec_expr(&arg, "null >= 0"), json!(true));
        assert_eq!(exec_expr(&arg, "null <= a"), json!(true));
    }
```

# PySQL

> The PySQL syntax is used in SQL to modify the SYNTAX of SQL and is a form of dynamic SQL

* py syntax, support for addition, subtraction, multiplication, and division if, for, in the trim, include, where, the
  set, choose, and so on syntax (and mybatis using the functions are almost the same)In the
* PY syntax, the line space for Child must be greater than that for father. It says it's its child
* PY syntax must end with:
* PY syntax support many express in same line.for example trim 'a': for item in arg:

| method    | rust code |
| ------ | ------ |
| trim 'AND ': | trim |
| if arg!=1 : | if |
| for item in arg : | for |
| set : | sql:"SET" |
| choose : | match |
| when : | match expr |
| otherwise : | match { _ =>{} } |
| _: | match { _ =>{} }(v1.8.54 later) |
| where : | sql:"WHERE" |
| bind a=1+1: | let a = 1+1 |
| let  a=1+1:  | let a = 1+1(v1.8.54 later)|

> example

```rust
    SELECT * FROM biz_activity
    if  name!=null:
      AND delete_flag = #{del}
      AND version = 1
      if  age!=1:
        AND version = 1
      AND version = 1
    AND a = 0
      yes
    for item in ids:
      #{item}
    for index,item in ids:
      #{item}
    trim 'AND':
      AND delete_flag = #{del2}
    choose:
        when age==27:
          AND age = 27
        otherwise:
          AND age = 0
    WHERE id  = '2';
```

> 1 Execute PysQL directly using Rbatis

``` rust
        let rb = Rbatis::new();
        rb.link("mysql://root:123456@localhost:3306/test").await.unwrap();
            let py = r#"
        SELECT * FROM biz_activity
        WHERE delete_flag = #{delete_flag}
        if name != null:
          AND name like #{name+'%'}
        if ids != null:
          AND id in (
          trim ',':
             for item in ids:
               #{item},
          )"#;
            let data: serde_json::Value = rb.py_fetch("", py, &json!({   "delete_flag": 1 })).await.unwrap();
            println!("{}", data);
```

> 2 Use Macro mapping to perform PysQL, see # Macro-Intelligent Macro mapping

# Mapper Macro

> Macros make it easy to write custom SQL, which is useful when you're writing complex multi-table associated queries, while keeping things simple and extensible. similar to @select dynamic SQL for Java/Mybatis

* sql macro:   Used to write raw SQL.   Rule: The first parameter to the SQL macro is the Rbatis instance name followed by SQL. Note that the SQL macro executes SQL
  that is driven to run directly, so it must be a replacement symbol for a specific database, such as mysql(? ,?) ,pg(
  $1,$2) for example ``` #[sql(RB, "select * from biz_activity where id = ?")] ```
* py_sql macro:  For writing 'dynamic SQL'.  Rule:   ```#{}``` is used instead of precompiled parameters (precompiled is
  safer and anti-SQL injection), and ```${}``` is used instead of direct replacement parameters (SQL injection risk).
* py_sql can use arithmetic expressions of macros, such as ` ` ` # {1 + 1}, # {arg}, # {arg [0]}, # {arg [0] + 'string'} ` ` `
* automatically converts the function ``` pub async fn select(name: & STR) -> rbatis::core::Result {} ```
* support Page Plugin!(Just put param ``` page_req: &PageRequest ```)
* param support``` tx_id: &str ``` or ``` context_id: &str ```  
* For PostgresSQL databases, precompiled SQL is used by default. Special types such as UUID require ::type cast type. For example, ' ' '#{arg}::uuid' ' '

> Macro mapping native driver SQL

```rust
    lazy_static! {
     static ref RB:Rbatis=Rbatis::new();
   }

    #[sql(RB, "select * from biz_activity where id = ?")]
    async fn select(name: &str) -> BizActivity {}

    #[async_std::test]
    pub async fn test_macro() {
        fast_log::log::init_log("requests.log", &RuntimeType::Std);
        RB.link("mysql://root:123456@localhost:3306/test").await.unwrap();
        let a = select("1").await.unwrap();
        println!("{:?}", a);
    }
```

> Macro mapping py_SQL (passed in schema referenced by Rbatis)

```rust
    lazy_static! {
     static ref RB:Rbatis=Rbatis::new();
   }

    #[py_sql(rbatis, "select * from biz_activity where id = #{name}
                  if name != '':
                    and name=#{name}")]
    async fn py_select(rbatis:&Rbatis,name: &str) -> Option<BizActivity> {}
   
    #[async_std::test]
    pub async fn test_macro_py_select() {
        fast_log::log::init_log("requests.log", &RuntimeType::Std);
        RB.link("mysql://root:123456@localhost:3306/test").await.unwrap();
        let a = py_select(&RB,"1").await.unwrap();
        println!("{:?}", a);
    }
```

> Macro mapping py_SQL (schema passing in transaction TX_ID)

```rust
    lazy_static! {
     static ref RB:Rbatis=Rbatis::new();
   }

    #[py_sql(RB, "select * from biz_activity where id = #{name}
                  if name != '':
                    and name=#{name}")]
    async fn py_select(tx_id:&str,name: &str) -> Option<BizActivity> {}
    #[async_std::test]
    pub async fn test_macro_py_select() {
        fast_log::log::init_log("requests.log", &RuntimeType::Std);
        RB.link("mysql://root:123456@localhost:3306/test").await.unwrap();
        let a = py_select("","1").await.unwrap();
        println!("{:?}", a);
    }
```

> Macro mapping py_SQL (join)

```rust
#[py_sql(rbatis, "SELECT a1.name as name,a2.create_time as create_time 
                  FROM test.biz_activity a1,biz_activity a2 
                  WHERE a1.id=a2.id and a1.name=#{name}")]
    async fn join_select(rbatis: &Rbatis, name: &str) -> Option<Vec<BizActivity>> {}

    #[async_std::test]
    pub async fn test_join() {
        fast_log::log::init_log("requests.log", &RuntimeType::Std);
        RB.link("mysql://root:123456@localhost:3306/test").await.unwrap();
        let results = join_select(&RB, "test").await.unwrap();
        println!("data: {:?}", results);
    }
```

> Macro mapping use page plugin

```rust
#[sql(RB, "select * from biz_activity where delete_flag = 0 and name = ?")]
async fn sql_select_page(page_req: &PageRequest, name: &str) -> Page<BizActivity> {}

#[py_sql(RB, "select * from biz_activity where delete_flag = 0
                  if name != '':
                    and name=#{name}")]
async fn py_select_page(page_req: &PageRequest, name: &str) -> Page<BizActivity> {}
```

> Disable generate Rust code print

```toml
rbatis-macro-driver = { version = "last version" , default-features=false, features = ["no_print"]}
```

# Conditional compilation choose Runtime or driver

> Conditional compilation can select the specified database and run-time compilation instead of compiling all databases. Conditional compilation can reduce program size
> Conditional compilation supports any of the following compilation parameters

|Options | explanation|
| ------ | ------ |
|default | when using async IO (async STD) runtime, all databases|
|async-io | when using async IO (async STD) runtime, all databases|
|actix | when using Actix runtime, all databases|
|tokio02 | when running with the version of tokio02, all databases|
|tokio03 | when running with the version of tokio03, all databases|
|async-io-mysql | when running with async STD version, MySQL database|
|async-io-postgres | when running with async STD version, PG database|
|async-io-sqlite | when using async STD version, SQLite database|
|async-io-mssql | when running with async STD version, MSSQL database|
|tokio03-mysql | when running with the tokio03 version, MySQL database|
|tokio03-postgres | using tokio03 version runtime, PG database|
|tokio03-sqlite |  Using tokio03 version runtime, SQLite database |
|tokio03-mssql | when running with the version of tokio03, MSSQL database|
|tokio02-mysql | when running with the version of tokio02, MySQL database|
|tokio02-postgres | using tokio02 version runtime, PG database|
|tokio02-sqlite | when using the tokio02 version runtime, SQLite database|
|tokio02-mssql | when running with the version of tokio02, MSSQL database|
|actix-mysql | when running with Actix version, MySQL database|
|actix-postgres | when running with Actix version, PG database|
|actix-sqlite | when running with Actix version, SQLite database|
|actix-mssql  | when running with Actix version, MSSQL database|
|tokio1-mysql | tokio1.0 mysql |
|tokio1-postgres | tokio1.0，PG database |
|tokio1-sqlite | tokio1.0，sqlite |
|tokio1-mssql | tokio1.0，mssql |
> for example,use 'actix-mysql'

```rust
rbatis = { version = "*", default-features = false, features = ["actix-mysql","snowflake"] }
```

# Plugin: RbatisPagePlugin

```rust
        let mut rb = Rbatis::new();
        rb.link("mysql://root:123456@localhost:3306/test").await.unwrap();
        //The framework defaults to the RbatisPagePlugin. If customization is needed, the structure must implement Impl PagePlugin for Plugin***{}, for example:
        //rb.page_plugin = Box::new(RbatisPagePlugin {});

        let req = PageRequest::new(1, 20);//分页请求，页码，条数
        let wraper= rb.new_wrapper()
                    .eq("delete_flag",1);
        let data: Page<BizActivity> = rb.fetch_page_by_wrapper("", &wraper,  &req).await.unwrap();
        println!("{}", serde_json::to_string(&data).unwrap());
```

> json result

```json
//2020-07-10T21:28:40.036506700+08:00 INFO rbatis::rbatis - [rbatis] Query ==> SELECT count(1) FROM biz_activity  WHERE delete_flag =  ? LIMIT 0,20
//2020-07-10T21:28:40.040505200+08:00 INFO rbatis::rbatis - [rbatis] Args  ==> [1]
//2020-07-10T21:28:40.073506+08:00 INFO rbatis::rbatis - [rbatis] Total <== 1
//2020-07-10T21:28:40.073506+08:00 INFO rbatis::rbatis - [rbatis] Query ==> SELECT  create_time,delete_flag,h5_banner_img,h5_link,id,name,pc_banner_img,pc_link,remark,sort,status,version  FROM biz_activity  WHERE delete_flag =  ? LIMIT 0,20
//2020-07-10T21:28:40.073506+08:00 INFO rbatis::rbatis - [rbatis] Args  ==> [1]
//2020-07-10T21:28:40.076506500+08:00 INFO rbatis::rbatis - [rbatis] Total <== 5
{
  "records": [
    {
      "id": "12312",
      "name": "null",
      "pc_link": "null",
      "h5_link": "null",
      "pc_banner_img": "null",
      "h5_banner_img": "null",
      "sort": "null",
      "status": 1,
      "remark": "null",
      "create_time": "2020-02-09T00:00:00+00:00",
      "version": 1,
      "delete_flag": 1
    }
  ],
  "total": 5,
  "size": 20,
  "current": 1,
  "serch_count": true
}
```

# transaction

## default transaction

```rust
#[async_std::test]
pub async fn test_tx() {
    fast_log::init_log("requests.log", 1000,log::Level::Info,true);
    let RB = Rbatis::new();
    RB.link(MYSQL_URL).await.unwrap();
    
    //let (tx_id,_)=rb.begin_tx().await.unwrap();//Since version 1.8.40 also you can use begin_tx()

    // Since version 1.8.39, the transaction format is'tx:transactionID',no ‘tx:’  the id at the beginning is executed in normal mode
    let tx_id = "tx:1";
    //begin
    RB.begin(tx_id).await.unwrap();
    let v: serde_json::Value = RB.fetch(tx_id, "SELECT count(1) FROM biz_activity;").await.unwrap();
    println!("{}", v.clone());
    //commit or rollback
    RB.commit(tx_id).await.unwrap();
}
```

## TxGuard

```rust
#[async_std::test]
    pub async fn test_tx_commit_defer() {
        fast_log::init_log("requests.log", 1000, log::Level::Info, None, true);
        let rb: Rbatis = Rbatis::new();
        rb.link(MYSQL_URL).await.unwrap();
        //use defer tx，You can forget to commit or roll back the transaction at the end of any function, and the framework will help you commit and roll back the transaction when the guard is reclaimed
        let guard = rb.begin_tx_defer(true).await.unwrap();
        let v: serde_json::Value = rb.fetch(&guard.tx_id, "SELECT count(1) FROM biz_activity;").await.unwrap();
        // tx will be commit
        drop(guard);
        println!("{}", v.clone());
        sleep(Duration::from_secs(1));
    }

2020-12-03 14:53:24.908263 +08:00    INFO rbatis::plugin::log - [rbatis] [tx:4b190951-7a94-429a-b253-3ec3df487b57] Begin
2020-12-03 14:53:24.909074 +08:00    INFO rbatis::plugin::log - [rbatis] [tx:4b190951-7a94-429a-b253-3ec3df487b57] Query ==> SELECT count(1) FROM biz_activity;
2020-12-03 14:53:24.912973 +08:00    INFO rbatis::plugin::log - [rbatis] [tx:4b190951-7a94-429a-b253-3ec3df487b57] ReturnRows <== 1
2020-12-03 14:53:24.914487 +08:00    INFO rbatis::plugin::log - [rbatis] [tx:4b190951-7a94-429a-b253-3ec3df487b57] Commit

```

## macro transaction

```rust
    #[py_sql(rbatis, "SELECT a1.name as name,a2.create_time as create_time
                      FROM test.biz_activity a1,biz_activity a2
                      WHERE a1.id=a2.id
                      AND a1.name=#{name}")]
    async fn join_select(rbatis: &Rbatis , tx_id:&str , name: &str) -> Option<Vec<BizActivity>> {}

    #[async_std::test]
    pub async fn test_join() {
        fast_log::log::init_log("requests.log", &RuntimeType::Std);
        RB.link("mysql://root:123456@localhost:3306/test").await.unwrap();

        //let (tx_id,_)=rb.begin_tx().await.unwrap(); //1.8.40 start using begin_tx() to return automatically generated TX_ID
        //The transaction format is'tx:transactionID', and non-tx: starting ids are executed in normal mode
        let tx_id = "tx:1";
        //begin
        RB.begin(tx_id).await.unwrap();
        let results = join_select(&RB, "test").await.unwrap();
        println!("data: {:?}",tx_id, results);
        //commit or rollback
        RB.commit(tx_id).await.unwrap();
    }
```


## transaction guard

```rust
#[async_std::test]
    pub async fn test_tx_commit_defer() {
        fast_log::init_log("requests.log", 1000, log::Level::Info, None, true);
        let rb: Rbatis = Rbatis::new();
        rb.link(MYSQL_URL).await.unwrap();
        //使用defer事务，你可以在任何函数结尾忘记提交或回滚事务，框架会在守卫被回收时帮助你提交，回滚事务
        let guard = rb.begin_tx_defer(true).await.unwrap();
        let v: serde_json::Value = rb.fetch(&guard.tx_id, "SELECT count(1) FROM biz_activity;").await.unwrap();
        // tx will be commit
        drop(guard);
        println!("{}", v.clone());
        sleep(Duration::from_secs(1));
    }

2020-12-03 14:53:24.908263 +08:00    INFO rbatis::plugin::log - [rbatis] [tx:4b190951-7a94-429a-b253-3ec3df487b57] Begin
2020-12-03 14:53:24.909074 +08:00    INFO rbatis::plugin::log - [rbatis] [tx:4b190951-7a94-429a-b253-3ec3df487b57] Query ==> SELECT count(1) FROM biz_activity;
2020-12-03 14:53:24.912973 +08:00    INFO rbatis::plugin::log - [rbatis] [tx:4b190951-7a94-429a-b253-3ec3df487b57] ReturnRows <== 1
2020-12-03 14:53:24.914487 +08:00    INFO rbatis::plugin::log - [rbatis] [tx:4b190951-7a94-429a-b253-3ec3df487b57] Commit

```

## transaction macro

```rust
    #[py_sql(rbatis, "SELECT a1.name as name,a2.create_time as create_time
                      FROM test.biz_activity a1,biz_activity a2
                      WHERE a1.id=a2.id
                      AND a1.name=#{name}")]
    async fn join_select(rbatis: &Rbatis , tx_id:&str , name: &str) -> Option<Vec<BizActivity>> {}

    #[async_std::test]
    pub async fn test_join() {
        fast_log::log::init_log("requests.log", &RuntimeType::Std);
        RB.link("mysql://root:123456@localhost:3306/test").await.unwrap();

        let tx_id = "1";
        //begin
        RB.begin(tx_id).await.unwrap();
        let results = join_select(&RB,tx_id, "test").await.unwrap();
        println!("data: {:?}", results);
        //commit or rollback
        RB.commit(tx_id).await.unwrap();
    }
```


# Conditional compilation switch runtime



> conditional compilation can select the specified database, run time compilation, and not compile the entire database. Conditional compilation can reduce program size
> conditional compilation supports any of the following compilation parameters (radio)

| single option | explains |
| ------ | ------ |
| default | - runs with async-io(async-std) on all drivers |
| async-io | uses async-io(async-std) run when all drivers |
| actix | using the actix runtime, all drivers |
| tokio02 | using the tokio02 version of the runtime, all drivers |
| tokio03 | using the tokio03 version of the runtime, all drivers |
| async-io-mysql | uses the async-std version run when mysql driver |
| async-io-postgres | using async-std version run, pg driver |
| async-io-sqlite | uses the async-std version run when sqlite driver |
| async-io-mssql | uses async-std version to run while MSSQL driver |
| tokio03-mysql | using the tokio03 version of the runtime, mysql driver |
| tokio03-postgres | using the tokio03 version of the run time, pg driver |
| tokio03-sqlite | using the tokio03 version of the run, sqlite driver |
| tokio03- MSSQL | using the tokio03 version of the run time, MSSQL driver |
| tokio02-mysql | uses the tokio02 version to run, mysql to driver |
| tokio02-postgres | uses the tokio02 version to run, pg driver |
| tokio02-sqlite | using the tokio02 version of the runtime, sqlite driver |
| tokio02-mssql | using the tokio02 version run time, MSSQL driver |
| actix-mysql | using the actix version of the runtime, mysql driver |
| actix-postgres | runs with the actix version, pg driver |
| actix-sqlite | using the actix version of the runtime, sqlite driver |
| actix-mssql | using the actix version runs, MSSQL driver |

> such as radio actix-mysql

```toml

rbatis = { version = "*", default-features = false, features = ["actix-mysql","snowflake"] }

` ` `



# Plugin: RbatisLogicDeletePlugin

> (Logical delete the query and delete methods provided for Rbatis are valid, such as fetch_list**(),remove**(), fetch**())

```rust
   let mut rb = init_rbatis().await;
   //rb.logic_plugin = Some(Box::new(RbatisLogicDeletePlugin::new_opt("delete_flag",1,0)));//Customize deleted/undeleted writing
   rb.logic_plugin = Some(Box::new(RbatisLogicDeletePlugin::new("delete_flag")));
   rb.link("mysql://root:123456@localhost:3306/test").await.unwrap();
           let r = rb.remove_batch_by_id::<BizActivity>("", &["1".to_string(), "2".to_string()]).await;
           if r.is_err() {
               println!("{}", r.err().unwrap().to_string());
   }
```

# Plug-in: SqlIntercept

> Implementing an interface

```rust
pub struct Intercept{}

impl SqlIntercept for Intercept{

    ///the intercept name
    fn name(&self) -> &str;

    /// do intercept sql/args
    /// is_prepared_sql: if is run in prepared_sql=ture
    fn do_intercept(&self, rb: &Rbatis, sql: &mut String, args: &mut Vec<serde_json::Value>, is_prepared_sql: bool) -> Result<(), rbatis::core::Error>;
}
```

# Plug-in: LogPlugin

```rust
use log::{debug, error, info, LevelFilter, trace, warn};
pub struct RbatisLog {}

impl LogPlugin for RbatisLog {
    
    //LevelFilter，allow close log print
    fn get_level_filter(&self) -> &LevelFilter {
        &self.level_filter
    }

    fn error(&self, data: &str) {
        error!("{}", data);
    }

    fn warn(&self, data: &str) {
        warn!("{}", data);
    }

    fn info(&self, data: &str) {
        info!("{}", data);
    }

    fn debug(&self, data: &str) {
        debug!("{}", data);
    }

    fn trace(&self, data: &str) {
        trace!("{}", data);
    }
}
```

> log plugin set into Rbatis

```rust
let mut rb=Rbatis::new();
rb.log_plugin = Box::new(RbatisLog{});
```

# Plug-in: distributed unique ID (snowflake algorithm)

```toml
rbatis = { version = "1.8", features = ["snowflake"] }
```

```rust
    use crate::plugin::snowflake::{async_snowflake_id, block_snowflake_id};

    #[test]
    fn test_new_async_id() {
        crate::core::runtime::block_on(async {
            println!("{}", async_snowflake_id().await);
        });
    }
```

> Set to Rbatis

```rust
let mut rb=Rbatis::new();
rb.sql_intercepts.push(Box::new(Intercept{}));
```


# Plug-in：Version lock/optimistic lock

> When updating a record, it is hoped that the record has not been updated by someone else
Optimistic locking implementation:

* Retrieves the current version when the record is fetched
* When updating, bring this version with you
* When updating, set version = newVersion(newVersion = oldVersion + 1) where version = oldVersion
* If the version is incorrect, it is not updated

```rust
 fast_log::init_log("requests.log", 1000, log::Level::Info, None, true);
 let mut rb = Rbatis::new();
 rb.version_lock_plugin = Some(Box::new(RbatisVersionLockPlugin::new("version")));
 rb.link("mysql://root:123456@localhost:3306/test").await.unwrap();
 let mut activity = BizActivity {
            id: Some("12312".to_string()),
            name: None,
            pc_link: None,
            h5_link: None,
            pc_banner_img: None,
            h5_banner_img: None,
            sort: None,
            status: Some(1),
            remark: None,
            create_time: Some(NaiveDateTime::now()),
            version: Some(BigDecimal::from(1)),
            delete_flag: Some(1),
        };
 let w = rb.new_wrapper().eq("id", "12312");
 let r = rb.update_by_wrapper("", &mut activity, &w, false).await;
 //[rbatis] [] Exec  ==> UPDATE biz_activity SET  status = ?, create_time = ?, version = ?, delete_flag = ? WHERE version = ? AND id = ?
 //[rbatis] [] Args  ==> [1,"2021-01-30T01:45:35.207863200","2",1,"1","12312"]
```

#### note:
- If the optimistic lock field is NULL, it will not work. If it is not NULL, it will work
- Supported data types are: i8,i32,i64... ,u32,u64... , string (integer or bigDecimal string) "i32" for example "0"..." 99999"
- Integer or string integer newVersion = oldVersion + 1
- newVersion is written back to entity!
- Only the Update * method is supported

# Contact information
WeChat ID: ``` zxj347284221  ```
WeChat group: add WeChat first, then pull in the group




