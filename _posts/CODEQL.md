```
codeql database create /Users/theoyu/workspace/cool/CodeQL/databases/micro-service-seclab-database -l java --command="mvn clean install --file pom.xml"
```

```
from [datatype] var
where condition(var = something)
select var
```