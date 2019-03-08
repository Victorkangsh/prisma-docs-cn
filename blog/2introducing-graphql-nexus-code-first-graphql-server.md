# GraphQL Nexus简介：代码优先的GraphQL服务器开发

> 翻译:[hanagm](https://github.com/hanagm)
>
> [原文链接](https://www.prisma.io/blog/introducing-graphql-nexus-code-first-graphql-server-development-ll6s1yy5cxl5/)





在[上一篇](./the-problems-of-schema-first-graphql-development.md)文章中，我们概述了SDL-first GraphQL服务器开发的问题。本周，我们很高兴地宣布GraphQL Nexus，这是一个代码优先的GraphQL库。邀请者[Tim Griesser](https://twitter.com/tgriesser)的文章。

![img](https://www.prisma.io/static/posts/introducing-graphql-nexus-code-first-graphql-server-development.png)



本文是由三部分组成的系列文章的*第二*部分：

- 第1部分：[“架构优先”GraphQL服务器开发的问题](./1the-problems-of-schema-first-graphql-development.md)
- 第2部分：**使用GraphQL Nexus进行代码优先的GraphQL服务器开发**
- 第3部分：[将GraphQL Nexus与数据库一起使用](./3using-graphql-nexus-with-a-database.md)

**在[Twitter上关注我们，](https://twitter.com/prisma)**以便在即将发布的文章时收到通知。



------

### 回顾：SDL首次开发的问题



如前一篇[文章所述](./the-problems-of-schema-first-graphql-development.md)，SDL-first GraphQL服务器开发存在许多挑战，例如*保持SDL和解析器同步*，*模块化GraphQL架构*以及*实现出色的IDE支持*。大多数问题*都*可以解决，但这只会以学习，使用和集成无数其他工具为代价。



今天我们正在介绍一个库，它实现了GraphQL服务器开发的代码优先方法：[**GraphQL Nexus**](https://nexus.js.org/)。

------

### 介绍GraphQL Nexus

#### 两全其美：架构优先和代码优先

在上一篇文章中，我们开发了对*schema-* first，*SDL-* first 和*code-* first方法的理解，用于构建GraphQL服务器：

- ***schema-first**：前端架构设计是开发过程的关键部分
- **SDL-first**：GraphQL架构的SDL版本是API *的真实来源*
- ***code-* first**：GraphQL架构以编程方式构建

作为代码优先框架，GraphQL Nexus仍可用于schema优先开发。架构优先和代码优先不是对立的方法：它们在组合时变得更有用。

使用Nexus，GraphQL架构以编程方式定义和实现。因此，它遵循GraphQL服务器在其他语言中的成熟方法，例如[`sangria-graphql`](https://github.com/sangria-graphql/sangria)（Scala）[`graphlq-ruby`](https://github.com/rmosolgo/graphql-ruby)或[`graphene`](https://graphene-python.org/)（Python）。

#### 类型安全，兼容GraphQL生态系统和数据无关

GraphQL Nexus在设计时考虑了TypeScript / JavaScript intellisense。它结合了TypeScript泛型，条件类型和类型合并，以提供完全自动生成的类型覆盖。Nexus的核心设计目标是使用尽可能少的手动类型注释来获得最佳类型覆盖。

![img](https://i.imgur.com/KMUm6rd.png)

Nexus基于其原语`graphql-js`，使其与当前的GraphQL生态系统基本兼容。



#### 使用Nexus定义和实现GraphQL架构

Nexus的API公开了许多函数，允许您为GraphQL架构定义和实现构建块，例如[**Object Types**](https://nexus.js.org/docs/api-objecttype)，[**Unions**](https://nexus.js.org/docs/api-uniontype)和[**Interfaces**](https://nexus.js.org/docs/api-interfacetype)，[**Enums**](https://nexus.js.org/docs/enumtype)以及您在[GraphQL类型系统中](https://facebook.github.io/graphql/draft/#sec-Type-System)找到的所有其他内容：

**Object Types**

```
const User = objectType({
  name: 'User',
  definition(t) {
    t.int('id', { description: 'Id of the user' })
    t.string('fullName', { description: 'Full name of the user' })
    t.list.field('posts', {
      type: Post, // or "Post"
      resolve(root, args, ctx) {
        return ctx.getUser(root.id).posts()
      },
    })
  },
})

const Post = objectType({
  name: 'Post',
  definition(t) {
    t.int('id')
    t.string('title')
  },
})
```

**Unions**

```
const MediaType = unionType({
  name: 'MediaType',
  description: 'Any container type that can be rendered into the feed',
  definition(t) {
    t.members('Post', 'Image', 'Card')
    t.resolveType(item => item.name)
  },
})
```

**Interfaces**

```
const Node = interfaceType({
  name: 'Node',
  definition(t) {
    t.id('id', { description: 'GUID for a resource' })
  },
})

const User = objectType({
  name: 'User',
  definition(t) {
    t.implements('Node')
  },
})
```

**Input Types**

```
const InputType = inputObjectType({
  name: 'InputType',
  definition(t) {
    t.string('key', { required: true })
    t.int('answer')
  },
})
```

**Enums**

```
// Definining as an array of enum values:
const Episode = enumType({
  name: 'Episode',
  members: ['NEWHOPE', 'EMPIRE', 'JEDI'],
  description: 'The first Star Wars episodes released',
})

// As an object, with a simple mapping of enum values to internal values:
const Episode = enumType({
  name: 'Episode',
  members: {
    NEWHOPE: 4,
    EMPIRE: 5,
    JEDI: 6,
  },
})
```

**Scalars**

```
const DateScalar = scalarType({
  name: "Date",
  asNexusMethod: "date"
  description: "Date custom scalar type",
  parseValue(value) {
    return new Date(value)
  },
  serialize(value) {
    return value.getTime()
  },
  parseLiteral(ast) {
    if (ast.kind === Kind.INT) {
      return new Date(ast.value)
    }
    return null
  },
})
```

该`Query`和`Mutation`类型是所谓的*根种*在GraphQL模式。Nexus提供了一个速记API来定义：

**Query**

```javascript
const Query = queryType({
  definition(t) {
    t.field('user', {
      type: User,
      nullable: true,
      args: { id: idArg({ nullable: false }) },
      resolve: (parent, { id }) => fetchUserById(id),
    })
  },
})
```

**Mutation**

```ts
const Mutation = mutationType({
  definition(t) {
    t.field('createUser', {
      type: User,
      args: { name: stringArg() },
      resolve: (parent, { name }) => createUser(name),
    })
  },
})
```

一旦为GraphQL架构定义了所有类型，就可以使用该[`makeSchema`](https://nexus.js.org/docs/api-makeschema)函数创建一个[`GraphQLSchema`](https://graphql.org/graphql-js/type/#graphqlschema)将成为GraphQL服务器基础的实例（例如`graphql-yoga`或`apollo-server`）：

```ts
const schema = makeSchema({
  // The programmatically defined building blocks of your GraphQL schema
  types: [User, Query, Mutation],

  // Specify where the generated TS typings and SDL should be located
  outputs: {
    typegen: __dirname + '/generated/typings.ts',
    schema: __dirname + '/generated/schema.graphql',
  },

  // All input arguments and return types are non-null by default
  nonNullDefaults: {
    input: true,
    output: true,
  },
})

// ... feed the `schema` into your GraphQL server (e.g. apollo-server or graphql-yoga)
```

`makeSchema`还可以让您提供[更漂亮的配置，](https://prettier.io/docs/en/configuration.html)以便生成的代码符合您的风格指南💅



### GraphQL Nexus入门

开始使用Nexus的最快方法是探索官方[示例](https://github.com/prisma/nexus/tree/develop/examples)或使用在线[Playground](https://nexus.js.org/playground)。

#### 1）安装

由于GraphQL Nexus严重依赖`graphql-js`，因此需要作为安装的[对等依赖项](https://nodejs.org/en/blog/npm/peer-dependencies/)：

**npm**

```bash
npm install --save nexus graphql
```

**yarn**

```bash
yarn add nexus graphql
```



#### 2）配置和最佳实践

文档中的[最佳实践](https://nexus.js.org/docs/best-practices)部分包含许多有关理想编辑器设置的说明以及构建Nexus项目的提示。



当graphql nexus动态生成类型时，开发人员最好的体验是通过在后台运行的开发服务器来实现的。无论何时保存文件，它都会更新生成的类型。



**为TypeScript配置开发服务器**

> 使用TypeScript时，一种可能的设置是[`ts-node-dev`](https://github.com/whitecolor/ts-node-dev)用于开发服务器：
>
> **npm**
>
> ```bash
> npm install -save -dev ts-node-dev
> ```
>
> **yarn**
>
> ```bash
> yarn add -D ts-node-dev
> ```
>
> 然后，您可以在以下位置配置用于开发的npm脚本`package.json`：
>
> ```json
> {
>   // ...
>   "scripts": {
>     "start": "...",
>     "dev": "ts-node-dev --no-notify --transpileOnly --respawn ./src"
>   }
> }
> ```

**为JavaScript配置开发服务器**

> 使用JavaScript时，您可以使用[`nodemon`](https://github.com/remy/nodemon)：
>
> **npm**
>
> ```bash
> npm install --save -dev nodemon
> ```
>
> **yarn**
>
> ```bash
> yarn add -D nodemon
> ```
>
> 然后，您可以在以下位置配置用于开发的npm脚本`package.json`：
>
> ```json
> {
>   // ...
>   "scripts": {
>     "start": "...",
>     "dev": "nodemon ./src/index.js"
>   }
> }
> ```

#### 3）基于 `graphql-yoga`的“Hello World”

完成编辑器设置后，就可以开始构建graphql模式了。下面是一个“Hello World”应用程序与`graphql-yoga`的代码片段：

```ts
import { queryType, stringArg, makeSchema } from 'nexus'
import { GraphQLServer } from 'graphql-yoga'

const Query = queryType({
  definition(t) {
    t.string('hello', {
      args: { name: stringArg({ nullable: true }) },
      resolve: (parent, { name }) => `Hello ${name || 'World'}!`,
    })
  },
})

const schema = makeSchema({
  types: [Query],
  outputs: {
    schema: __dirname + '/generated/schema.graphql',
    typegen: __dirname + '/generated/typings.ts',
  },
})

const server = new GraphQLServer({
  schema,
})

server.start(() => `Server is running on http://localhost:4000`)
```

#### 4）从SDL-first API迁移

在[SDL转换器](https://nexus.js.org/converter)可让您提供的SDL模式定义，并输出相应的Nexus代码（没有任何 resolvers）：

![img](https://imgur.com/AbkFWNO.png)

### 努力获得出色的开发者体验

Nexus API的设计特别注重开发人员的体验。一些核心设计目标是：

- 况下类型安全

- 可读性

- 开发人员工效学

- 与[Prettier](https://prettier.io/)轻松集成


在构建API时运行的开发服务器可确保您始终对刚刚引入的架构更改进行自动完成和错误检查。

借助GraphQL Playground中的新[架构轮询功能](https://medium.com/novvum/6c9da4bbd552)，您可以在调整架构时立即重新加载GraphQL API。





### 让我们知道你的想法✍️



我们对[GraphQL Nexus](https://github.com/prisma/nexus/)非常兴奋，并希望你也会如此。您可以通过浏览官方[示例](https://github.com/prisma/nexus/tree/develop/examples)或按照文档中的[“入门”指示](https://nexus.js.org/docs/getting-started)随意试用Nexus 。



如果您遇到任何问题，请[打开GitHub问题](https://github.com/prisma/nexus/issues/new)或在我们的[Slack中联系](https://slack.prisma.io/)。