---
layout: post
title:  "Scripting SQL Server Databases"
date:   2009-10-28 19:33:27
categories: misc
---

This post first appeared on the [LShift web site](http://www.lshift.net/blog/2009/10/28/scripting-sql-server-databases/).

I recently had a need to move a SQL Server database around several development environments. I would normally use SQL Server Management Studio to generate creation and drop scripts, but as the database was changing frequently, I wanted a way of automating this process. Being able to script the structure and data separately was also useful, as the latter changed more frequently and independently than the former.

We use PowerShell for the bulk of the admin tasks around the project, so this, in conjunction with the [SMO (SQL Server Management Objects) API](http://msdn.microsoft.com/en-us/library/ms162169.aspx) was a natural starting point. The basic script was easy, but the problem came in working out dependencies so that the generated scripts could be executed without barfing on constraint problems. For our project I went with the quick and dirty solution of disabling integrity checks whilst performing the operations, i.e.:

{% highlight sql %}
EXEC sp_MSForEachTable 'ALTER TABLE NOCHECK CONSTRAINT ALL'
{% endhighlight %}

Whilst this achieved my aim, it was a bit of a cop-out. I later managed to get something closer to what I had originally envisaged (essentially “mysqldump” but the ability to script only the data if requested). This is how I got there.

The first step was to load the SMO assembly, and create a Scripter object:a

    [reflection.assembly]::LoadWithPartialName("Microsoft.SqlServer.Smo") | Out-Null
    $global:serverSmo = New-Object ('Microsoft.SqlServer.Management.Smo.Server') 'DAVESMOBILESQLEXPRESS'
    $global:scrp = New-Object ('Microsoft.SqlServer.Management.Smo.Scripter') ($serverSmo)

I used the classic Microsoft sample “NorthWind” database to test with, so a reference to that was needed as well:

    $global:db = $serverSmo.Databases['NorthWind']

I started looking at automated generation by dropping all tables so I could be confident that old properties were not left lingering. This was straightforward in theory:

    function global:Generate-DropTables() {
        $scrp.Options.ScriptDrops = $True
        $scrp.Options.IncludeIfNotExists = $True    
        $scrp.Script($db.Tables)
    }

Sadly, executing the generated script failed because it does not take into account the relationships between the tables. Enabling scripting of DRI (Declarative Referential Integrity) attributes by using the `$scrp.Options.DriAll` option produced a script where the referential integrity constraints are dropped before the table.

This worked on the default Northwind database but will not work if any non-table object, e.g., a schema-bound view, depends on a table that is being dropped. The Scripting object does provide the option `$scrp.Options.WithDependencies` that when enabled, will generate DROP statements for any object that depends on a table. Using this worked well.

The method of creating tables is almost identical, except that any indexes, referential constraints and triggers should also be created so a few more options are needed:

    function global:Generate-CreateTables() {
        $scrp.Options.ScriptDrops = $False
        $scrp.Options.IncludeIfNotExists = $True
        
        $scrp.Options.ClusteredIndexes = $True
        $scrp.Options.DriAll = $True
        $scrp.Options.Indexes = $True
        $scrp.Options.Triggers = $True
    
        $scrp.Options.WithDependencies = $True
    
        $scrp.Script($db.Tables)
    }

Although this worked in and of itself, it is not complete when considered in the context of the previous drop function. When dropping tables, dependencies are objects that depend on the table and therefore must be dropped before we can drop the table. When creating tables, dependencies are objects that the table depends on and therefore must be created before the table can be created. These are not the same objects and so the drop script will drop objects that will not be re-created, and vice-versa.

Fortunately, help is at hand in the form of the DependencyWalker class. Using this, we can obtain a tree of references to objects that either depend or are dependent on a specified object. The DependencyWalker class also has a convenience method for obtaining an ordered list representation of the tree, producing a list of object references in the form of URNs:

    $depWalker = New-Object ('Microsoft.SqlServer.Management.Smo.DependencyWalker') $serverSmo
    $depTree = $depWalker.DiscoverDependencies($db.Tables, $True)
    $orderedUrns = $depWalker.WalkDependencies($depTree)
    
    foreach($urn in $orderedUrns) {
        $smoObject = $serverSmo.GetSmoObject($urn.Urn)
        $scrp.Script($smoObject)
    }

This is fine and dandy, but led on to how all objects can be scripted. After much mucking about with typing (does the SMO API really need *two* URN classes), assembly searching, and the never helpful “Discover dependencies failed” message, it turned out that discovering dependencies only works for a limited subset of objects; Views, Stored Procedures, Tables, User Defined Functions, and possibly a couple of others that did not exist in the database I was playing with.

Passing all the supported objects, filtered to exclude system objects, to the Scripter worked perfectly and resulted in what appeared a fairly complete script for the database structure.

I assumed following the same strategy for data would be sufficient to complete the process. After all, the `Scripter.Options` class has a `ScriptData` property, so surely it should just work, right? Wrong. The `Script()` method does not support data generation to prevent the data set “from being scripted to a StringCollection object”. The `EnumScript()` method should be used instead, which returns an enumeration over the script strings. I’m not sure how this is an improvement (memory? But surely there must be a source structure for the enumeration somewhere?), but there you go. I made the change and duly got a script to delete all the data and one to insert it all.

Now, there is a table in the NorthWind database with a foreign key that refers to itself. This means the` INSERT INTO` statements will need to be generated in the correct order. They were not. Setting `WithDependencies` to `$True` looked to be a promising route to follow, but this resulted in the INSERTs being generated twice. It appeared this actually stopped the foreign key problem, but I’ve no idea how. Spending time looking into this unexpected success didn’t seem worthwhile given there were now plenty of primary key violations to contend with.

In the end, I failed to come up with a solution to script the data independently of the structure. Instead I have produced a combined script that does both. It is possible to combine the generation of table creation statements and data inserts, and because check constraints are added at the end of the scripting of each object, then the ordering of row inserts is not a problem. Obviously, the tables must be non-existent / empty at this point otherwise there will be primary key problems so the DROP stage must be included as well.

I’ve attached the [resulting script](/assets/scripting-sql-server-db/dump-db.txt). It can be invoked (once renamed to Dump-Db.ps1) along the lines of:

    [reflection.assembly]::LoadWithPartialName("Microsoft.SqlServer.Smo") | Out-Null
    $serverSmo = New-Object ('Microsoft.SqlServer.Management.Smo.Server') 'MRGREEDYSQLEXPRESS'
    $db = $serverSmo.Databases['NorthWind']
    
    .Dump-Db.ps1
    
    Dump-Db $db > output.sql

