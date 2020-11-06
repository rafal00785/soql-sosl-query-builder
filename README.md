# SOQL and SOSL query builder

Build dynamic SOQL and SOSL queries.

## References

- [SOQL and SOQL.Builder class examples](#SOQL-and-SOQLBuilder-class-examples)
  - [Different ways to query fields](#Different-ways-to-query-fields)
  - [Relationship queries](#Relationship-queries)
  - [Scope filter](#Scope-filter)
  - [Condition Expression Syntax](#Condition-Expression-Syntax)
  - [Field Expression Syntax](#Field-Expression-Syntax)
  - [Records Order](#Records-Order)
  - [Scope limit](#Scope-limit)
  - [FOR keyword syntax](#FOR-keyword-syntax)
  - [Salesforce Article Keywords](#Salesforce-Article-Keywords)
- [SOQLAgregate and SOQLAgregate.Builder class examples](#SOQLAgregate-and-SOQLAgregateBuilder-class-examples)
  - [Aggregate Functions](#Aggregate-Functions)
  - [GROUP BY option](#GROUP-BY-option)
  - [Aliasing support](#Aliasing-support)
  - [HAVING option](#HAVING-option)
- [SOSL and SOSL.Builder class examples](#SOSL-and-SOSLBuilder-class-examples)
  - [Example Text Searches](#Example-Text-Searches)
  - [Condition Expression Syntax with the RETURNING clause](#Condition-Expression-Syntax-with-the-RETURNING-clause)
## SOQL and SOQL.Builder class examples
Use SOQL.Builder class to build your dynamic query. Use SOQL class to execute the query.
### Different ways to query fields
If no field is specified, the query will return only the **Id** field.
```Apex
Set<String> fieldsToSelect = new Set<String>{
    'AccountNumber', 'AccountSource', 'Phone'
};
Schema.FieldSet fieldSet = SObjectType.Account.FieldSets.Test;
SOQL soqlQuery = new SOQL.Builder('Account')
    .selectField('Name')
    .selectFields(fieldsToSelect)
    .selectFields(fieldSet)
    .selectField('FORMAT(Type)')
    .build();

//SELECT Name, AccountNumber, AccountSource, Phone, Id, OwnerId, FORMAT(Type) FROM Account
```
### Relationship queries
Child-to-parent.
```Apex
SOQL soqlQuery = new SOQL.Builder('Contact')
    .selectField('Account.Name')
    .build();

//SELECT Account.Name FROM Contact
```
Parent-to-child.
```Apex
SOQL soqlQuery = new SOQL.Builder('Account')
    .addSubQuery(
        new SOQL.Builder('Contacts')
            .selectField('ReportsTo.Name')
            .build()
    )
    .build();

//SELECT (SELECT ReportsTo.Name FROM Contacts) FROM Account
```
Polymorphic relationships.
```Apex
SOQL soqlQuery = new SOQL.Builder('Case')
    .addTypeOf(
        new Query.TypeOf('Owner')
            .whenSObjectType('User', new Set<String>{'Id', 'Username'})
            .whenSObjectType('Group', new Set<String>{'Id', 'DeveloperName'})
            .elseFieldList('Id')
    )
    .build();

//SELECT TYPEOF Owner WHEN User THEN Id, Username WHEN Group THEN Id, DeveloperName ELSE Id END FROM Case
```
### Scope filter
```Apex
SOQL soqlQuery = new SOQL.Builder('Account')
    .addScope(Query.Scope.Mine)
    .build();

//SELECT Id FROM Account USING SCOPE Mine
```
### Condition Expression Syntax
The default logical operator is **AND**.
```Apex
SOQL soqlQuery = new SOQL.Builder('Account')
    .whereCondition(
        new Query.Condition()
            .equals('Name', 'Test')
            .equals('Type', 'Other')
    )
    .build();

//SELECT Id FROM Account WHERE (Name = 'Test' AND Type = 'Other')
```
Define the logical operator value.
```Apex
SOQL soqlQuery = new SOQL.Builder('Account')
    .whereCondition(
        new Query.Condition(Query.LogicalOperator.OR_VALUE)
            .equals('Name', 'Test')
            .equals('Type', 'Other')
    )
    .build();

//SELECT Id FROM Account WHERE (Name = 'Test' OR Type = 'Other')
```
Subcondition support.
```Apex
SOQL soqlQuery = new SOQL.Builder('Account')
    .whereCondition(
        new Query.Condition(Query.LogicalOperator.OR_VALUE)
            .equals('Name', 'Test')
            .subcondition(
                new Query.Condition()
                    .notEquals('Type', 'Other')
                    .greaterThan('NumberOfEmployees', 0)
            )
    )
    .build();

//SELECT Id FROM Account WHERE (Name = 'Test' OR (Type != 'Other' AND NumberOfEmployees > 0))
```
Negate subcondition.
```Apex
SOQL soqlQuery = new SOQL.Builder('Account')
    .whereCondition(
        new Query.Condition()
            .negateSubcondition(
                new Query.Condition()
                    .notEquals('Type', 'Other')
                    .greaterThan('NumberOfEmployees', 0)
            )
    )
    .build();

//SELECT Id FROM Account WHERE (NOT(Type != 'Other' AND NumberOfEmployees > 0))
```

### Field Expression Syntax

#### Comparison Operators
Comparison operators, such as =, !=, <, <=, >, >=, LIKE, IN, NOT IN can be used in a SOQL query.
```Apex
Set<String> typesToSelect = new Set<String>{
    'Prospect', 'Other'
};
SOQL soqlQuery = new SOQL.Builder('Account')
    .whereCondition(
        new Query.Condition()
            .equals('NumberOfEmployees', 1000)
            .isLike('Name', 'Te%')
            .isIn('Type', typesToSelect)
    )
    .build();

//SELECT Id FROM Account WHERE (NumberOfEmployees = 1000 AND Name LIKE 'Te%' AND Type IN ('Prospect', 'Other'))
```
#### Querying Multi-Select Picklists
You can search for individual values in multi-select picklists
Includes. Get the list of records with the **MSP1\_\_c** field that values are equal to **AAA** and **BBB** or **CCC** selected (exact match).
```Apex
Set<Set<String>> valuesToSelect = new Set<Set<String>>{
    new Set<String>{'AAA', 'BBB'},
    new Set<String>{'CCC'}
};
SOQL soqlQuery = new SOQL.Builder('Account')
    .whereCondition(
        new Query.Condition()
            .includes('MSP1__c', valuesToSelect)
    )
    .build();

//SELECT Id FROM Account WHERE (MSP1__c INCLUDES ('AAA;BBB', 'CCC'))
```
Excludes. Get the list of records with the **MSP1\_\_c** field that values are not equal to **AAA** and **BBB** or **CCC** selected (exact match).
```Apex
Set<Set<String>> valuesToIgnore = new Set<Set<String>>{
    new Set<String>{'AAA', 'BBB'},
    new Set<String>{'CCC'}
};
SOQL soqlQuery = new SOQL.Builder('Account')
    .whereCondition(
        new Query.Condition()
            .excludes('MSP1__c', valuesToIgnore)
    )
    .build();

//SELECT Id FROM Account WHERE (MSP1__c EXCLUDES ('AAA;BBB', 'CCC'))
```
#### Supported Value types
Primitive. Query builder supports all primitive data types (Integer, Double, Long, Date, Datetime, String, ID, or Boolean).
```Apex
Integer numberOfEmployees = 1000;
SOQL soqlQuery = new SOQL.Builder('Account')
    .whereCondition(
        new Query.Condition()
            .equals('NumberOfEmployees', numberOfEmployees)
    )
    .build();

//SELECT Id FROM Account WHERE (NumberOfEmployees = 1000)
```
Collections. Query builder supports Lists and Sets for all primitive data types (Integer, Double, Long, Date, Datetime, String, ID, or Boolean).
```Apex
Set<Integer> numberOfEmployees = new Set<Integer>{1000, 2000};
SOQL soqlQuery = new SOQL.Builder('Account')
    .whereCondition(
        new Query.Condition()
            .isIn('NumberOfEmployees', numberOfEmployees)
    )
    .build();

//SELECT Id FROM Account WHERE (NumberOfEmployees IN (1000, 2000))
```
SOQL and SOQL.Builder objects. Query builder supports SOQL and SOQL.Builder objects to build semi join queries.
```Apex
SOQL soqlQuery = new SOQL.Builder('Account')
    .whereCondition(
        new Query.Condition()
            .isIn('OwnerId', new SOQL.Builder('Contact')
                .selectField('CreatedById')
            )
    )
    .build();

//SELECT Id FROM Account WHERE (OwnerId IN (SELECT CreatedById FROM Contact))
```
Date literals. Query builder supports date literals.
```Apex
SOQL soqlQuery = new SOQL.Builder('Account')
    .whereCondition(
        new Query.Condition()
            .equals('CreatedDate', new Query.DateLiteral('LAST_N_DAYS', 100))
    )
    .build();

//SELECT Id FROM Account WHERE (CreatedDate = LAST_N_DAYS:100)
```
Script variable. Query builder supports including script variables into the query string.
```Apex
Set<Integer> numberOfEmployees = new Set<Integer>{1000, 2000};
SOQL soqlQuery = new SOQL.Builder('Account')
    .whereCondition(
        new Query.Condition()
            .equals('NumberOfEmployees', new Query.ScriptVariable('numberOfEmployees'))
    )
    .build();
Database.query(soqlQuery.getQueryString());

//SELECT Id FROM Account WHERE (NumberOfEmployees = :numberOfEmployees)
```
### Records Order
Order the list of records by order direction and null records order.
```Apex
SOQL soqlQuery = new SOQL.Builder('Account')
    .orderBy(new Query.SortOrder('Type', Query.SortDirection.DESCENDING, Query.SortNullRecords.LAST))
    .build();

//SELECT Id FROM Account ORDER BY Type DESC NULLS LAST
```
### Scope-limit
Query the specific number of records.
```Apex
SOQL soqlQuery = new SOQL.Builder('Account')
    .orderBy(new Query.SortOrder('Type'))
    .setScopeLimit(10)
    .setOffset(10)
    .build();

//SELECT Id FROM Account ORDER BY Type LIMIT 10 OFFSET 10
```
### FOR keyword syntax
Get the set of records and update its reference view data.
```Apex
SOQL soqlQuery = new SOQL.Builder('Account')
    .setForReference()
    .build();

//SELECT Id FROM Account FOR REFERENCE
```
Get the set of records and locks them for other updates.
```Apex
SOQL soqlQuery = new SOQL.Builder('Account')
    .setForUpdate()
    .build();

//SELECT Id FROM Account FOR UPDATE
```
Get the set of records and updates its last view data.
```Apex
SOQL soqlQuery = new SOQL.Builder('Account')
    .setForView()
    .build();

//SELECT Id FROM Account FOR VIEW
```
### Salesforce Article Keywords
Update an Article’s Keyword Tracking with SOQL
```Apex
SOQL soqlQuery = new SOQL.Builder('FAQ__kav')
    .whereCondition(
        new Query.Condition()
            .equals('Keyword', 'Apex')
            .equals('Language', 'en_US')
            .equals('KnowledgeArticleVersion', 'ka230000000PCiy')
    )
    .setUpdateTracking()
    .build();

//SELECT Id FROM FAQ__kav WHERE (Keyword = 'Apex' AND Language = 'en_US' AND KnowledgeArticleVersion = 'ka230000000PCiy') UPDATE TRACKING
```
Update an Article Viewstat with SOQL
```Apex
SOQL soqlQuery = new SOQL.Builder('FAQ__kav')
    .whereCondition(
        new Query.Condition()
            .equals('PublishStatus', 'online')
            .equals('Language', 'en_US')
            .equals('KnowledgeArticleVersion', 'ka230000000PCiy')
    )
    .setUpdateViewStat()
    .build();

//SELECT Id FROM FAQ__kav WHERE (PublishStatus = 'online' AND Language = 'en_US' AND KnowledgeArticleVersion = 'ka230000000PCiy') UPDATE VIEWSTAT
```
## SOQLAgregate and SOQLAgregate.Builder class examples
Use SOQLAgregate.Builder class to build your dynamic agregate query. Use SOQLAgregate class to execute the query.
### Aggregate Functions
Use SOQLAgregate.Builder methods to include aggregate functions like AVG(), COUNT(), MIN(), MAX(), SUM(), and more to the agregate query.
```Apex
SOQLAgregate soqlAgregateQuery = new SOQLAgregate.Builder('Opportunity')
    .average('Amount')
    .build();

//SELECT AVG(Amount) FROM Opportunity
```
### GROUP BY option
The fields used to group results are automatically added to the returned field list.
```Apex
SOQLAgregate soqlAgregateQuery = new SOQLAgregate.Builder('Opportunity')
    .count('Name')
    .groupBy('LeadSource')
    .build();

//SELECT COUNT(Name), LeadSource FROM Opportunity GROUP BY LeadSource
```
### Aliasing support
Use Query.Condition to filter the query results.
```Apex
SOQLAgregate soqlAgregateQuery = new SOQLAgregate.Builder('Opportunity')
    .maximum('Id', 'max')
    .groupBy('Name')
    .build();

//SELECT MAX(Id) max, Name FROM Opportunity GROUP BY Name
```
### HAVING option
Use Query.Condition to filter the query results.
```Apex
SOQLAgregate soqlAgregateQuery = new SOQLAgregate.Builder('Opportunity')
    .count('Id')
    .groupBy('LeadSource')
    .havingCondition(
        new Query.Condition()
            .greaterThan('COUNT(Id)', 2)
    )
    .build();

//SELECT COUNT(Id), LeadSource FROM Opportunity GROUP BY LeadSource HAVING (COUNT(Id) > 2)
```
## SOSL and SOSL.Builder class examples
Use SOSL.Builder class to build your dynamic query. Use SOSL class to execute the query.
### Example Text Searches
Simple text search.
```Apex
String searchPhrase = 'Joe Smith';
SOSL soslQuery = new SOSL.Builder(searchPhrase)
    .build();

//FIND {Joe Smith}
```
Conditional text search.
```Apex
Query.SearchCondition searchCondition = new Query.SearchCondition(QUERY.LogicalOperator.OR_VALUE)
    .find('Joe Smith')
    .find('Joe Smythe');
SOSL soslQuery = new SOSL.Builder(searchCondition)
    .build();

//FIND {"Joe Smith" OR "Joe Smythe"}
```
Specify the information to be returned in the text search result.
```Apex
Query.ReturningFieldSpec returningFieldSpec = new Query.ReturningFieldSpec(Account.SObjectType)
    .selectFields(new Set<String>{'Id', 'Name'});
SOSL soslQuery = new SOSL.Builder('Joe Smith')
    .addReturningFieldSpec(returningFieldSpec)
    .build();

//FIND {Joe Smith} RETURNING Account (Id, Name)
```
### Condition Expression Syntax with the RETURNING clause
Limit the search results using condition expression
```Apex
SOSL soslQuery = new SOSL.Builder('Joe Smith')
    .addReturningFieldSpec(
        new Query.ReturningFieldSpec(Account.SObjectType)
            .selectField('Id')
            .whereCondition(
                new Query.Condition()
                    .equals('Reason', 'Other')
            )
    ).build();

//FIND {Joe Smith} RETURNING Account (Id WHERE (Reason = 'Other'))
```
Limit the search results using list view
```Apex
SOSL soslQuery = new SOSL.Builder('Joe Smith')
    .addReturningFieldSpec(
        new Query.ReturningFieldSpec(Account.SObjectType)
            .selectField('Id')
            .usingListView('Recent')
    ).build();

//FIND {Joe Smith} RETURNING Account (Id USING LISTVIEW = Recent)
```
Limit the search results using LIMIT and OFFSET syntax
```Apex
SOSL soslQuery = new SOSL.Builder('Joe Smith')
    .addReturningFieldSpec(
        new Query.ReturningFieldSpec(Account.SObjectType)
            .selectField('Id')
            .scopeLimit(10)
            .offset(0)
    ).build();

//FIND {Joe Smith} RETURNING Account (Id LIMIT 10 OFFSET 0)
```