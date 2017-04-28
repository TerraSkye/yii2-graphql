# yii-graphql #

Use Facebook [GraphQL](http://facebook.github.io/graphql/) and [React Relay](https://facebook.github.io/relay/). Extended [graphql-php](https://github.com/webonyx/graphql-php to apply to YII2, also part of the current development . Through this document [documentation](https://facebook.github.io/relay/docs/graphql-relay-specification.html#content) you can learn some knowledge of Relay.

At the same time on the decryption of Graphql-php design ideas derived from [laraval-graphql](https://github.com/Folkloreatelier/laravel-graphql).

Why should deconstruct graphql-php, the library is mainly to make the realization of the graphql protocol in php, do not need to consider how the actual project with better efficiency and performance. Including laraval-graphql is the same, from the ideal production applications Gap. As needed to define and lazy load are not implemented, will be abnormal configuration file maintenance is too difficult.

Yii-graphql mainly in the use of convenience and loading to do a lot of optimization.
1. Configuration to minimize, including simplified standard graphql protocol definition.
2. As needed \ lazy load, according to the type of defined full name, to achieve on-demand Load and lazy, do not need to be in the system at the beginning of all types of definition to load into.
3. If with the type of activerecord, relay for the table query operation is very simple.


todo
1.ActiveRecordType implementation of the navigation properties.
2.ActiveRecordType query schema definition
3.only for activerecordType due to the need to query the database definition, so the type of cache, consider the stress test to see whether the whole type of cache (minimum priority Level) 
4. return of error validation

some special syntax for graphql, like parameter syntax, interface syntax, built-in instruction syntax has not yet been tested

### installation ###

Install trough composer
```php    
"require": {
    "yiisoft/yii-graphql": "dev-master"
}
```

### Used in YII ###

Join in the modules of the yii configuration file

```php
    'graphql'=>$graphConfigFile
```

In the config configuration folder to add graph.php, configuration examples are as follows:



```php
retrun [
    'class'=>'yii\graphql\module'
   // main graphql protocol configuration
    'schemas' => [        
        'query' => [
            'user' => 'App\GraphQL\Query\UsersQuery'
        ],
        'mutation' => [

        ],
        //
        'types'=>[
            'user'=>'app\modules\graph\type\UserType'
        ],
    ],
     // cache key, the system uses the default cache configuration , But in practice the use of local file cache
    'cache'=>'cache',
];
```

In the case of dynamic analysis, if you do not want to define the types, schema can be used to write. You can use Type:: class, to avoid the use of Key methods, but also easy to navigate directly through the IDE to the corresponding class
```php
    'type'=>GraphQL::type(UserType::class)
```

### Type ###

Type system is the core of GraphQL, reflected in the GraphQLType, through the deconstruction of graphql protocol, and the use of graph-php library to achieve the fine-grained control of all elements to facilitate their own needs to expand the class.

GraphQLType the main elements, ** Note that the elements do not correspond to the attributes or methods (the same below) **

element  | Types of | Description
----- | ----- | -----
name | string | **Required** Each type needs to be named, and if it is unique, it is not mandatory, and the attribute needs to be defined in the attribute
fields | array | **Required**  contains the contents of the field, to fields () method.
resolveField | callback | **function($value, $args, $context, GraphQL\Type\Definition\ResolveInfo $info)** 对于字段的解释,比如fields定义user属性,则对应的解释方法为resolveUserField() ,$value指定为type定义的类型实例

### Query ###

GraphQLQuery, GraphQLMutation inherited the GraphQLField, the element structure is consistent, want to do for some reusable Field, you can inherit it. Graphql each query need to correspond to a GraphQLQuery object

The main element of the GraphQLField

 元素 | 类型  | 说明
----- | ----- | -----
type | ObjectType | 对应的查询类型,单一类型用GraphQL::type指定,列表用Type::listOf(GraphQL::type)
args | array | 查询需要使用的参数,其中每个参数按照Field定义
resolve | callback | **function($value, $args, $context, GraphQL\Type\Definition\ResolveInfo $info)**,$value为root数据,$args即查询参数,$context上下文,为Yii的yii\web\Application对象,$info为查询解析对象,一般在这个方法中处理根对象

### Mutation ###

与GraphQLQuery是非常相像,参考说明.

### 简化处理 ###

简化了Field的声明,字段可直接使用type

```php
标准方式
    'id'=>[
        'type'=>type::id(),
    ],
简化写法
    'id'=>type::id()
```



### Demo ###

#### 创建基于graphql协议的查询 ####

每次查询对应一个GraphQLQuery文件,
```php

class UserQuery extends GraphQLQuery
{
    public function type()
    {
        return GraphQL::type(UserType::class);
    }

    public function args()
    {
        return [
            'id'=>[
                'type'=>Type::nonNull(Type::id())
            ],
        ];
    }

    public function resolve($value, $args, $context, ResolveInfo $info)
    {
        return DataSource::findUser($args['id']);
    }


}
```

根据查询协议定义类型文件
```php

class UserType extends GraphQLType
{
    protected $attributes = [
        'name'=>'user',
        'description'=>'user is user'
    ];

    public function fields()
    {
        $result = [
            'id' => ['type'=>Type::id()],
            'email' => Types::email(),
            'email2' => Types::email(),
            'photo' => [
                'type' => GraphQL::type(ImageType::class),
                'description' => 'User photo URL',
                'args' => [
                    'size' => Type::nonNull(GraphQL::type(ImageSizeEnumType::class)),
                ]
            ],
            'firstName' => [
                'type' => Type::string(),
            ],
            'lastName' => [
                'type' => Type::string(),
            ],
            'lastStoryPosted' => GraphQL::type(StoryType::class),
            'fieldWithError' => [
                'type' => Type::string(),
                'resolve' => function() {
                    throw new \Exception("This is error field");
                }
            ]
        ];
        return $result;
    }

    public function resolvePhotoField(User $user,$args){
        return DataSource::getUserPhoto($user->id, $args['size']);
    }

    public function resolveIdField(User $user, $args)
    {
        return $user->id.'test';
    }

    public function resolveEmail2Field(User $user, $args)
    {
        return $user->email2.'test';
    }


}
```

#### 创建基于yii activerecord查询 ####

像普通的Query创建一个Query文件,只需要将type指向模型文件即可
```php
class UserModelQuery extends GraphQLQuery
{
    public function type()
    {
        return Type::listOf(GraphQL::type(UserModel::class));
    }

    public function args()
    {
        return [
            'id'=>[
                'type'=>Type::id(),
                'description'=>'用户的ID'
            ],
            'pageIndex'=>[
                'type'=>Type::int(),
                'description'=>''
            ],
            'pageSize'=> [
                'type'=> Type::int(),
                'description'=>'',
            ],
        ];
    }

    public function resolve($root,$args,$context,ResolveInfo $info){
        $id = $args['id'];
        return UserModel::findAll(['id'=>$id]);
    }

}
```

#### 查询实例 ####

```php
'hello' =>  "
        query hello{hello}
    ",

    'singleObject' =>  "
        query user {
            user(id:\"2\") {
                id
                email
                email2
                photo(size:ICON){
                    id
                    url
                }
                firstName
                lastName

            }
        }
    ",
    'multiObject' =>  "
        query multiObject {
            user(id: \"2\") {
                id
                email
                photo(size:ICON){
                    id
                    url
                }
            }
            stories(after: \"1\") {
                id
                author{
                    id
                }
                body
            }
        }
    ",
    'updateObject' =>  "
        mutation updateUserPwd{
            updateUserPwd(id: \"1001\", password: \"123456\") {
                id,
                username
            }
        }
    "
```

### 深入了解 ###

有必要了解一些graphql-php的相关知识,这部分git上的文档相对还少些,需要对源码的阅读.下面列出重点

#### DocumentNode (语法解构) ####

```
array definitions
    array OperationDefinitionNode
        string kind
        array NameNode
            string kind
            string value
```
