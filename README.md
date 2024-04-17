# hibernate6-h2-like-fix
Fix for bug in Hibernate 6 appending `escape ''` to LIKE conditions when using H2's Oracle compatibility mode.

The `org.hibernate.dialect.H2Dialect#visitLikePredicate` function appends the literal `escape ''` to the like predicate if the query itself does not specify an escape character. This explicitly contravenes the behavior of its parent class, {@link org.hibernate.sql.ast.spi.AbstractSqlAstTranslator}, which only appends ` escape ` to the query if the escape character is NOT null. As a result, using H2Dialect in Hibernate 6 will ALWAYS result in an escape clause being appended to a query that uses the like predicate.

In H2 Oracle compatibility mode, `escape ''` causes a SQL error because the escape character is not allowed to be an  empty string - directly contradicting the comment in H2Dialect that indicates the whole reason for overriding the 'escape'  clause is to prevent H2's default escape character of backslash being used. This creates an unnecessary problem because the default is EXACTLY what we expect to be used if no explicit 'escape' clause is included in the query!

A query with `escape ''` will silently error and this causes zero results to be returned although any reasonable person would expect the SQL error to be bubbled up as an exception that could be caught and managed.

For example, this Hibernate query:

```
from Dog where name LIKE '%Hamil%'
```

results in the generated SQL:

```
select * from dog where name like '%Hamil%' escape ''
```

None of this would be a problem if H2Dialect didn't override AbstractSqlAstTranslator's visitLikePredicate() function. However, we still need the rest of the functionality provided by H2Dialect, and it's not possible to set a global escape character that H2Dialect would respect or cause H2Dialect.visitLikePredicate()'s 'escape' clause handling to be skipped (and if any escape expression object is supplied, AbstractSqlAstTranslator ALWAYS appends the escape clause). The solution used here is to subclass H2Dialect, override visitLikePredicate() to supply a new LikePredicate with a hardcoded escape character matching the default (backslash), and provide our own output for the escape clause's value.

This custom dialect will still append `escape` to your LIKE queries, but it sets the escape character to the default (`escape '\'`) instead of the illegal `escape ''`.

## Usage

1. Drop the `H2OracleDefaultEscapeTranslator` and `H2OracleDefaultEscapeDialect` classes into your project
2. Set the `hibernate.dialect` property to `com.sperkins.hibernate.H2OracleDefaultEscapeDialect` in persistence.xml or wherever you set up your database properties
