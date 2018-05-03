# Transactions with MySQL and PDO

When you start with MySQL, you don't worry too much about whether all the queries you're going to make will be executed perfectly. You can of course put a `<? php or exit (mysql_error ());? >` at the end of each query (or a more elaborate system) but that is not always enough.

Indeed it is necessary that the MySQL server returns an error (so if it breaks down at this time, it is crazy) or that the PHP server can also treat this error (and like its colleague it can also have problems). It is therefore necessary to make sure that for some queries (INSERT and UPDATE among others) everything went as intended before applying the changes in the database.

> The goal of transactions is precisely to ensure that all these changes have been made correctly before they are definitively implemented. And in case of problems, transactions allow you to go back in time.

## Explanations

### Examples to better understand

> This first example is there to show you that in some cases it is essential that all queries are perfectly made.

#### Les transactions bancaires

We are going to take an honest citizen (Mr. Michou) who wants to buy a server that is worth 2000€ that he decides to pay by credit card. He gives his credit card number and all the information that goes with it so that the bank can transfer money between Mr. Michou's account and the account of the server seller. This bank, which after making sure that everything was in order, will then transfer the money from the buyer's account to the seller's account.

First of all the bank debits 2000€ from Mr Michou's account.

`UPDATE compte SET argent = argent - 2000 WHERE compte_proprio = michou`

Then she sends this money to the account of the server vendor.

`UPDATE compte SET argent = argent + 2000 WHERE compte_proprio = vendeur`

After the transfer was successfully completed, Mr. Michou received his server. Everything went well there because both requests went well, but there may also be an error during the transfer. Let's use the previous example:

First of all the bank debits 2000€ from Mr Michou's account.

`UPDATE compte SET argent = argent - 2000 WHERE compte_proprio = michou`

Then she sends this money to the account of the server vendor.

`// MySQL server crash`

And it's a tragedy (for Mr. Michou), the bank withdrew 2000€ from his account, but this money never arrived on the seller's account, so he will never send the server since he didn't receive any money. It should therefore be possible to ensure that these two requests have been carried out correctly before applying them. This is the purpose of this tutorial.

## MySQL storage engines

The particularity of MySQL is to offer several storage engines in the same database. Without going into the details, I will give you a brief presentation of some of these current engines by detailing a little bit their particularities. To understand when and why to use a particular storage engine. Because these engines have very special characteristics that have even led to misconceptions for a while.

### MyIsam: the most common

#### Benefits

MyIsam is the default engine of MySQL, if until now the notion of storage engine was unknown to you, it is probably this engine that you have been using from the beginning with MySQL. This engine is very popular because it is very easy to use (for a beginner) and offers very good performances on tables that are frequently opened in read-write mode.

Its other strong point is to propose a FULL-TEXT index, which makes it possible to make quite precise searches (compared to LIKE) on columns of text and which allows each one to have a small search engine, in particular thanks to a sorting by relevance.

#### Cons

To stay on the FULL-TEXT index, MyIsam suffers nevertheless on the big tables, moreover its configuration (word size in particular) is accessible only on a dedicated server.

But the 2 biggest flaws of the MyIsam engine is that it does not support neither foreign keys, nor transactions (which are the subject of this tutorial). Moreover, it is the strong presence of this engine (more proposed by default) which makes some people believe that MySQL does not manage transactions. That's not true, but you have to use a storage engine that supports them: InnoDB.

### InnoDB: an engine for robust databases

Contrary to MyIsam, InnoDB is an engine that is used for its functionalities which make it the most used engine in sensitive sectors, i. e. requiring a coherence and a great integrity of the data (finance, online games, complex architecture very much in demand, etc...).

Its two main strengths are its management of foreign keys and its support of transactions. These transactional mechanisms are highly compatible with the ACID criteria.

Concerning its defects: besides having larger tables (on average 25% larger) and not offering a FULL-TEXT index, InnoDB is slightly slower in operations, but this is due to integrity tests (foreign keys and transactions) which allow to keep a consistent basis.

> So it is this storage engine that we will use for the rest of this tutorial.

### Memory (Heap): everything in Ram

As its name indicates, this storage engine stores the table data in memory, the structure when it is stored in a file. Its main interest is its speed of access, very useful for a table in high demand. The problem is that in the event of a server shutdown, all stored data is deleted (since it is stored in the Ram which empties when the power is turned off).

It is therefore necessary to use this storage engine for data that is not essential to the functioning of a site such as a visitor counter or chat system (unless you want to keep a record of the discussions).

Storage engines for MySQL, there are plenty of others but the main ones are there, let's move on to practice without further delay.

## Theory: MySQL

As mentioned above, you have to use the InnoDB storage engine, so you have to specify it when creating the table you are going to work on:

```
CREATE TABLE compte
(
    -- row lists
)
ENGINE = InnoDB;
```
This very simple table that we will use, using the first example, has 2 fields:
* the name of the account owner
* the amount on the account

```
CREATE TABLE compte
(
    nom VARCHAR (30) NOT NULL,
    montant MEDIUMINT UNSIGNED NOT NULL
)
ENGINE = InnoDB;
```
We will of course insert our 2 users, the seller and Mr. Michou (here called buyer) by giving them each a account with money enough.
```
INSERT INTO compte
(nom,montant)
VALUES
('vendeur', '10000'),
('acheteur', '25000');
```

### Transactions: COMMIT and ROLLBACK

The principle of transactions is very simple to implement: we start the transaction with START TRANSACTION; then we carry out the operations on the tables, and then we have 2 choices:
* Previous operations are validated with `COMMIT`.
* All changes are cancelled with `ROLLBACK`.

By default, however, the InnoDB engine is set to automatically validate all transactions because it considers each individual request as a transaction: we cancel this behavior thanks to the command `SET autocommit = 0;`, we will see immediately what it gives in practice:
```
-- we disable autocommit
SET autocommit = 0;
-- we start transaction
START TRANSACTION;
-- we make this request on our table compte
UPDATE compte SET montant = montant + 20000 WHERE nom = 'vendeur';
```
You can make this request sequence as many times as you want, the seller's amount will not change by an inch, the transaction has not been validated. To do this, you must call the `COMMIT` command which will validate the requests made during the transaction:
```
-- we disable autocommit
SET autocommit = 0;
-- we start transaction
START TRANSACTION;
-- we make this request on our table compte
UPDATE compte SET montant = montant + 20000 WHERE nom = 'vendeur';
-- we valid transaction
COMMIT;
```
And as if by magic, the request is taken into account.

There is the `COMMIT` command to validate, but there is also the `ROLLBACK` command to cancel, this makes it possible to return to the structure before the beginning of the transaction, in other words we make a Ctrl + Z on our database. Very useful if there is a problem like this for example:
```
-- we disable autocommit
SET AUTOCOMMIT =0;
-- we start transaction
START TRANSACTION;
-- we make this request on our table compte
UPDATE compte SET montant = montant + 20000 WHERE nom = 'vendeur';
-- this request returns an error (the user 'machin' doesn't exist)
UPDATE compte SET montant = montant - 20000 WHERE nom = 'machin';
-- we cancel the transaction
ROLLBACK ;
```
If these two operations had been carried out outside a transaction, the seller would have received his money but there would have been a problem since the member does not exist, so in the end the total amount in my table is no longer the same, we then have more a coherent basis from where the importance of the transactions.

In order to make the best use of transactions, we will now move on to examples using PHP.

## Practice: Using PDO

### PDO as interface with MySQL

#### PDO Introduction

There are many ways to connect to a database with PHP, one of them is to use PDO which makes it possible to use transactions natively.

PDO uses the concept of object-oriented programming. The first thing to do is to connect to the database: just like with the usual MySQl driver, you need to provide the following information:
* The database address (localhost if you are working locally)
* The name of the database (here transactions)
* The username (here root)
* the password to access the database (here test)

All that remains is to create a PDO object to interact with MySQL.

`$pdo = new PDO('mysql:host=localhost;dbname=transactions', 'root', 'test');`

#### Exceptions

One of the advantages of PDO is the ability to use exceptions. This will allow us to launch a transaction and depending on the result (failure or success of the transaction) act accordingly:

* If successful: continue the rest of the script (confirmation, validation, etc.).
* In case of failure: an error has been detected, it is then possible to retrieve it (in a log for example) and then process it. Most often the errors detected are foreign key problems or simply coding errors. In this sense, exceptions are also very useful to debug a system without ruining everything during testing.

```php
try
{
    $pdo = new PDO('mysql:host=localhost;dbname=transactions', 'root', 'test');
}
catch(Exception $e)
{
    echo 'Failure to connect to database';
    exit();
}
```

The first part can be seen as an attempt by the system to execute the code placed in the first bracket. If all goes well then everything is fine but if the script portion returns an error, then it will be able to be used by the `catch` command. Then the code placed in the brackets will be executed. In our example we will then display the error in order to know why the connection to the database failed.

```
catch(Exception $e)
{
    echo 'Error: '.$e->getMessage().'<br />';
    echo 'Code#: '.$e->getCode();
}
```

Most of the time the error is explicit (bad password, the database doesn't exist, the SQL server doesn't respond), otherwise with the given error cdoe and a search on Google we find the solution very quickly.

#### A complete transaction

The first step is to initialize the transaction with the PDO `startTransaction` method (it has rarely been simpler).

`$pdo->beginTransaction();`

The next steps are very easy. You should simply execute the requests you want (these ones are fancy)

```
$pdo->query("SELECT * FROM machin WHERE bidule = 'truc'");
$pdo->query("INSERT INTO machin SET bidule = 'truc', chose = 'moi'");
$pdo->query("UPDATE machin SET nombre = nombre + 1");
```
Finally we confirm all these requests with `commit` method:

`$pdo->commit();`

If all goes well then these 3 queries will be applied to database, but in case of error we will treat it and cancel previous queries with `rollback` method.

`$pdo->rollback();`

Which with the system of exceptions gives this kind of structure:

```php
try
{  //we try to execute the following requests in a transaction scheme

    //we start transaction
    $pdo->beginTransaction();

    // our 3 requests
    $pdo->query("SELECT * FROM machin WHERE bidule = 'truc'");
    $pdo->query("INSERT INTO machin SET bidule = 'truc', chose = 'moi'");
    $pdo->query("UPDATE machin SET nombre = nombre + 1");

    // if everything is ok, we validate transaction
    $pdo->commit();

    //we display a confirmation message
    echo 'Everything is valid.';
}
catch(Exception $e) //in case of error
{
    //we cancel transaction
    $pdo->rollback();

    //we display an error message and the errors
    echo 'There is an error<br>';
    echo 'Error message: '.$e->getMessage().'<br>';
    echo 'Code: '.$e->getCode();

    //we stop execution of the rest of the code
    exit();
}
```

So that's what a complete transaction looks like, it makes it possible to be sure that everything went well and that we won't end up with an erroneous database like for example a message from a forum that isn't attached to any topic because the creation of the topic failed.
