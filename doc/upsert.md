# Upsert

upsert block包含了一个query block和多个mutation blocks，query block中定义的变量可以通过val（）和uid（）在mutation blocks中使用。

query block中的变量可以定义多个uid，并使用uid来接收他们。执行query block时，有两种可能的结果：

1）：如果查询条件匹配不到任何节点，name返回的变量就是空的，uid（）会返回一个新的uid，类似于空白节点。对于删除操作，它不会返回uid，并且这个操作会默认被忽略，不会执行。

2）：如果变量中存储了多个uid，那么uid（）函数会将所有uid返回，并对所有uid执行操作。

### Example of uid Function

定义schema如下：

```
name: string @index(term) .
email: string @index(exact, trigram) @upsert .
age: int @index(int) .
```

假如我们要创建一个包含电子邮件和姓名的新用户，我们需要确保一个电子邮件在数据库中只能对应一个用户。因此，我们必须先查询数据库中是否存在与给定电子邮件对应的用户，如果用户存在，就使用该查询到的uid更新信息，如果不存在，就创建一个新用户。

upsert如下：

```
upsert {
  query {
    q(func: eq(email, "user@company1.io")) {
      v as uid
      name
    }
  }

  mutation {
    set {
      uid(v) <name> "first last" .
      uid(v) <email> "user@company1.io" .
    }
  }
}
```

将查询到的uid给变量v，如果v存在，则更新信息；如果不存在，则uid（v）被认为是空白节点，并将信息插入到数据库中。

json格式的upsert语法如下：

```
{
  "query": "{ q(func: eq(email, \"user@company1.io\")) {
  	v as uid, name
  } 
  }",
  "set": {
    "uid": "uid(v)",
    "name": "first last",
    "email": "user@company1.io"
  }
}
```

### Bulk Delete Example

upsert也可以用于批量删除

如果想从数据库中删除“company1”的所有用户，使用upsert只需要一次查询就可以实现

```
{"query": "{ 
	v as var(func: regexp(email, /.*@company1.io$/)) 
	}",
  "delete": {
    "uid": "uid(v)",
    "name": null,
    "email": null,
    "age": null
  }
}
```

### Example of val Function

在upsert中，除了可以查询uid作为变量，也可以查询其它谓词的值作为变量

如果想要获取age属性的值，并将其作为其它谓词的属性，比如：

```
upsert {
  query {
    v as var(func: has(age)) {
      a as age
    }
  }

  mutation {
    # we copy the values from the old predicate
    set {
      uid(v) <other> val(a) .
    }

    # and we delete the old predicate
    delete {
      uid(v) <age> * .
    }
  }
}
```

变量a会存储所有uid与age的映射，然后upsert block会在other中存储每个uid对应的age值，并删除age谓词。

相当于统一替换了key，将所有值转移到新的key中

json格式如下：

```
{"query": "{ v as var(func: regexp(email, /.*@company1.io$/)) }",
  "delete": {
    "uid": "uid(v)",
    "age": null
  },
  "set": {
    "uid": "uid(v)",
    "other": "val(a)"
  }
}
```

### External IDs and Upsert Block

使用upsert也可以很容易地管理外部ID

假如有如下schema：

```
xid: string @index(exact) .
<http://schema.org/name>: string @index(exact) .
<http://schema.org/type>: [uid] @reverse .
```

初始化data：

```
{
  set {
    _:blank <xid> "http://schema.org/Person" .
    _:blank <dgraph.type> "ExternalType" .
  }
}
```

现在可以使用upsert block创建一个新的person，并扩充它的type

```
   upsert {
      query {
        var(func: eq(xid, "http://schema.org/Person")) {
          Type as uid
        }
        var(func: eq(<http://schema.org/name>, "Robin Wright")) {
          Person as uid
        }
      }
      mutation {
          set {
           uid(Person) <xid> "https://www.themoviedb.org/person/32-robin-wright" .
           uid(Person) <http://schema.org/type> uid(Type) .
           uid(Person) <http://schema.org/name> "Robin Wright" .
           uid(Person) <dgraph.type> "Person" .
          }
      }
    }
```

