# Practice and use of PDO

### Establish a connection with PDO

The first thing to do to use PDO is of course to establish a database connection:

```
try {
    $strConnection = 'mysql:host=localhost;dbname=ma_base'; //Ligne 1
    $arrExtraParam= array(PDO::MYSQL_ATTR_INIT_COMMAND => "SET NAMES utf8");
    $pdo = new PDO($connStr, 'Utilisateur', 'Mot de passe', $arrExtraParam); // Instancie la connexion
    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION); //Ligne 4
}
catch(PDOException $e) {
    $msg = 'ERREUR PDO dans ' . $e->getFile() . ' L.' . $e->getLine() . ' : ' . $e->getMessage();
    die($msg);
}
```

There are three types of error:

1. **PDO::ERRMODE_SILENT** - does not report errors (but will assign error codes);
2. **PDO::ERRMODE_WARNING** - triggers a warning;
3. **PDO::ERRMODE_EXCEPTION** - triggers an exception.

### Without prepared statements
We can use the methods query() and exec().
In practice:
1. You use **`query()`** for select requests **(SELECT)**
2. and **`exec()`** to insert data **(INSERT)**, update data **(UPDATE)** or delete data **(DELETE)**.

**Example - Make a `query` and a `fetch`**.
- *PDO Version*
```
$query = 'SELECT * FROM foo WHERE bar=1;';
$arr = $pdo->query($query)->fetch(); // Sur une même ligne ...
```

- *MySQL Version*
```
$query = 'SELECT * FROM foo WHERE bar=1;';
$result = mysql_query($query);
$arr = mysql_fetch_assoc($result);
```
**Example - Make a `query` and a `fetchAll` :**
- *PDO Version*
```
$query = 'SELECT * FROM foo WHERE bar<10;';
$stmt = $pdo->query($query);
$arrAll = $stmt->fetchAll(); //... ou sur 2 lignes
```
- *MySQL Version*
```
$query = 'SELECT * FROM foo WHERE bar<10;';
$result = mysql_query($query);
$arrAll = array();
while($arr = mysql_fetch_assoc($result))
      $arrAll[] = $arr;
```
**Example - Make an `exec` :**
- *PDO Version*
```
$query = 'DELETE FROM foo WHERE bar<10;';
$rowCount = $pdo->exec($query);
```

- *MySQL Version*
```
$query = 'DELETE FROM foo WHERE bar<10;';
mysql_query($query);
$rowCount = mysql_affected_rows();
```

### With prepared statements
We will now see how to make prepared statements with PDO, compared to mysqli\_ and mysql\_.

**Example - Make a `prepare` et un `fetchAll`**
- *PDO Version*

```
// Prepare request
$query = 'SELECT *'
	. ' FROM foo'
	. ' WHERE id=?'
		. ' AND cat=?'
	. ' LIMIT ?;';
$prep = $pdo->prepare($query);

// Match values with holders
$prep->bindValue(1, 120, PDO::PARAM_INT);
$prep->bindValue(2, 'bar', PDO::PARAM_STR);
$prep->bindValue(3, 10, PDO::PARAM_INT);

// Compile et execute the request
$prep->execute();

// Retrieve all returned data
$arrAll = $prep->fetchAll();

// Close the prepared request
$prep->closeCursor();
$prep = NULL;
```

- *MySQL Version*

```
// Prepare the request
$query = 'PREPARE stmt_name'
	. ' FROM "SELECT *'
		. ' FROM foo'
		. ' WHERE id=?'
			. ' AND cat=?'
		. ' LIMIT ?";';
mysql_query($query);

// Match values with holders
$query = 'SET @paramId = 120;';
mysql_query($query);
$query = 'SET @paramCat = "bar";';
mysql_query($query);
$query = 'SET @paramLimit = 10;';
mysql_query($query);

// Compile et execute the request
$query = 'EXECUTE stmt_name'
	. ' USING @paramId,'
		. ' @paramCat,'
		. ' @paramLimit;'
$result = mysql_query($query);

// Retrieve all returned data
$arrAll = array();
while($arr = mysql_fetch_assoc($result))
     $arrAll[] = $arr;

// Close the prepared request
$query = 'DEALLOCATE PREPARE stmt_name;';
mysql_query($query);
```
### Reuse a prepared statement to gain performance
The initial advantage of prepared statements is the reuse of the statement template.
Indeed, the DBMS has already processed a part of the request.
It is therefore possible to re-run the query with new values, without having to start processing again; the cutting and interpretation has already been done!

**Reuse a prepared statement**
```
$query = 'INSERT INTO foo (nom, prix) VALUES (?, ?);';
$prep = $pdo->prepare($query);

$prep->bindValue(1, 'item 1', PDO::PARAM_STR);
$prep->bindValue(2, 12.99, PDO::PARAM_FLOAT);
$prep->execute();

$prep->bindValue(1, 'item 2', PDO::PARAM_STR);
$prep->bindValue(2, 7.99, PDO::PARAM_FLOAT);
$prep->execute();

$prep->bindValue(1, 'item 3', PDO::PARAM_STR);
$prep->bindValue(2, 17.94, PDO::PARAM_FLOAT);
$prep->execute();

$prep = NULL;
```

## ATTENTION
When using prepared statements, you can only associate values and not SQL commands.
