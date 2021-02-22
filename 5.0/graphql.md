# GraphQL

## 快速开始

GraphQL 是一种用于 API 的查询语言，是使用现有数据来完成这些查询的运行时。这是一种优雅的方法，可以解决我们在典型REST apis 中遇到的许多问题 。这里是 GraphQL 和 REST 之间一个很好的[比较](https://dev-blog.apollodata.com/graphql-vs-rest-5d425123e34b) 。在这组文章中, 我们不会解释什么是 GraphQL, 而是演示如何使用 `@nestjs/GraphQL` 模块。

GraphQLModule 只不过是 [Apollo](https://www.apollographql.com) 服务器的包装器。我们没有造轮子, 而是提供一个现成的模块, 这让 GraphQL 和 Nest 有了比较简洁的融合方式。


### 安装

首先，我们需要安装所需的软件包：

```bash
$ npm i --save @nestjs/graphql apollo-server-express graphql-tools graphql
```

译者注： fastify 请参考：
https://github.com/coopnd/fastify-apollo

### 入门

一旦安装了软件包，我们就可以注册`GraphQlModule`

> app.module.ts

```typescript
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';

@Module({
  imports: [
    GraphQLModule.forRoot({
      typePaths: ['./**/*.graphql'],
    }),
  ],
})
export class ApplicationModule {}
```

`typePaths` 属性指示 `GraphQLModule` 应在何处查找 GraphQL 文件。此外, 所有选项都将传递到基础的 Apollo 实例 (在[这里](https://www.apollographql.com/docs/apollo-server/v2/api/apollo-server.html#constructor-options-lt-ApolloServer-gt)阅读更多的可用设置)。例如, 如果要禁用 `playground` 并关闭 `debug` 模式, 只需通过以下选项:

> app.module.ts

```typescript
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';

@Module({
  imports: [
    GraphQLModule.forRoot({
      typePaths: ['./**/*.graphql'],
      debug: false,
      playground: false,
    }),
  ],
})
export class ApplicationModule {}
```

如上所述, 所有这些设置都将转发到 ApolloServer 构造函数。

### Playground

Playground 是一个图形化的，交互式的浏览器内 GraphQL IDE，默认情况下可与 GraphQL 服务器本身 URL 相同。当您的应用程序在后台运行时，打开 Web 浏览器并访问： http://localhost:3000/graphql （主机和端口可能因您的配置而异）。

![](https://docs.nestjs.com/assets/playground.png)

### 多个端点

该模块的另一个有用功能是能够同时为多个端点提供服务。多亏了这一点，您可以决定哪个模块应该包含在哪个端点中。默认情况下，`GraphQL` 在整个应用程序中搜索解析器。要仅限制模块的子集，可以使用该 `include` 属性。

```typescript
GraphQLModule.forRoot({
  include: [CatsModule],
})
```

### Async 配置


通常, 您可能希望异步传递模块选项, 而不是预先传递它们。在这种情况下, 使用 `forRootAsync()` 函数, 它提供了处理异步数据的几种不同方法。

第一种可能的方法是使用工厂功能:

```typescript
GraphQLModule.forRootAsync({
  useFactory: () => ({
    typePaths: ['./**/*.graphql'],
  }),
})
```
显然, 我们的 factory 的行为和其他人一样 (可能是异步的, 并且能够通过 `inject` 注入依赖关系)。

```typescript
GraphQLModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: async (configService: ConfigService) => ({
    typePaths: configService.getString('GRAPHQL_TYPE_PATHS'),
  }),
  inject: [ConfigService],
})
```

或者, 您可以使用类而不是 factory。

```typescript
GraphQLModule.forRootAsync({
  useClass: GqlConfigService,
})
```
上面的构造将实例化 `GqlConfigService` 内部 `GraphQLModule`, 并将利用它来创建选项对象。`GqlConfigService` 必须实现 `GqlOptionsFactory` 接口。

```typescript
@Injectable()
class GqlConfigService implements GqlOptionsFactory {
  createGqlOptions(): GqlModuleOptions {
    return {
      typePaths: ['./**/*.graphql'],
    };
  }
}
```

为了防止 `GqlConfigService`内部创建 `GraphQLModule` 并使用从不同模块导入的提供程序，您可以使用 `useExisting` 语法。


```typescript
GraphQLModule.forRootAsync({
  imports: [ConfigModule],
  useExisting: ConfigService,
})
```

它的工作原理与 `useClass` 有一个关键的区别—— `GraphQLModule` 将查找导入的模块重用已经创建的 `ConfigService`, 而不是单独实例化它。

### 例子

[这里](https://github.com/nestjs/nest/tree/master/sample/12-graphql-apollo)这里提供完整的案例。

## 解析图


通常，您必须手动创建解析图。 `@nestjs/graphql` 包也产生解析器映射，可以自动使用由装饰器提供的元数据。
为了学习库基础知识，我们将创建一个简单的用户 API。首先，让我们在 SDL 中定义我们的类型（阅读[更多](http://graphql.org/learn/schema/#type-language)）：


```graphql
type Author {
  id: Int!
  firstName: String
  lastName: String
  posts: [Post]
}

type Post {
  id: Int!
  title: String
  votes: Int
}

type Query {
  author(id: Int!): Author
}
```

我们的 GraphQL 架构包含公开的单个查询 `author(id: Int!): Author` 。现在，让我们创建一个 `AuthorResolver` 。

```graphql
@Resolver('Author')
export class AuthorResolver {
  constructor(
    private readonly authorsService: AuthorsService,
    private readonly postsService: PostsService,
  ) {}

  @Query()
  async author(@Args('id') id: number) {
    return await this.authorsService.findOneById(id);
  }

  @ResolveProperty()
  async posts(@Parent() author) {
    const { id } = author;
    return await this.postsService.findAll({ authorId: id });
  }
}
```

?> 提示：如果使用 `@Resolver()` 装饰器，则不必将类标记为 `@Injectable()` ，否则必须这么做。


`@Resolver()` 装饰器不影响查询和对象变动 (`@Query()` 和 `@Mutation()` 装饰器)。这只会通知 Nest, 每个 `@ResolveProperty()` 有一个父节点, `Author` 在这种情况下是父节点。

通常, 我们会使用像 `getAuthor()` 或 `getPosts()` 之类的函数来命名。我们可以很容易地做到这一点, 以及移动命名之间的装饰器的括号。

```typescript
@Resolver('Author')
export class AuthorResolver {
  constructor(
    private readonly authorsService: AuthorsService,
    private readonly postsService: PostsService,
  ) {}

  @Query('author')
  async getAuthor(@Args('id') id: number) {
    return await this.authorsService.findOneById(id);
  }

  @ResolveProperty('posts')
  async getPosts(@Parent() author) {
    const { id } = author;
    return await this.postsService.findAll({ authorId: id });
  }
}
```

?> 这个 `@Resolver()` 装饰器可以在函数级别被使用。

### 装饰

在上面的示例中，您可能会注意到我们使用专用装饰器来引用以下参数。下面是提供的装饰器和它们代表的普通Apollo参数的比较。

|||
|---|---|
| @Root() 和 @Parent() |	root/parent |
| @Context(param?:string)	| context/context[param] |
| @Info(param?:string)	| info/info[param] |
| @Args(param?:string)	| args/args[param] |

### Module

一旦我们在这里完成，我们必须在 `AuthorResolver` 的某处注册，例如在新创建的 `AuthorsModule` 内部注册。

```typescript
@Module({
  imports: [PostsModule],
  providers: [AuthorsService, AuthorResolver],
})
export class AuthorsModule {}
```
该 `GraphQLModule` 会考虑反映了元数据和转化类到正确的解析器的自动映射。您应该注意的是您需要在某处 import 此模块，因此 Nest 将知道 `AuthorsModule` 确实存在。

?> 提示：在[此处](http://graphql.cn/learn/queries/)了解有关 GraphQL 查询的更多信息。

### 类型

单独创建 GraphQL 类型和相应的 TypeScript 定义会产生不必要的冗余。最终，我们最终没有这样做，SDL 内部的每个变化都会迫使我们修改接口。因此，该`@nestjs/graphql` 包提供了另一个有趣的功能，即使用抽象语法树（AST）自动生成 TS 定义。要启用它，只需自定义 definitions 属性即可。

```typescript
GraphQLModule.forRoot({
  typePaths: ['./**/*.graphql'],
  definitions: {
    path: join(process.cwd(), 'src/graphql.ts'),
  },
})
```
该 `src/graphql.ts` 指示在何处保存 typescript 输出。默认情况下，所有类型都转换为接口。但是，您可以通过将 outputAs 属性更改为 class。

```typescript
GraphQLModule.forRoot({
  typePaths: ['./**/*.graphql'],
  definitions: {
    path: join(process.cwd(), 'src/graphql.ts'),
    outputAs: 'class',
  },
})
```

因此，它将生成以下文件：

```typescript
export class Author {
  id: number;
  firstName?: string;
  lastName?: string;
  posts?: Post[];
}

export class Post {
  id: number;
  title?: string;
  votes?: number;
}

export abstract class IQuery {
  abstract author(id: number): Author | Promise<Author>;
}
```

类允许您使用装饰器，这使得它们在验证方面非常有用（阅读[更多](/5.0/techniques?id=验证)）。例如：

```typescript
import { MinLength, MaxLength } from 'class-validator';

export class CreatePostInput {
  @MinLength(3)
  @MaxLength(50)
  title: string;
}
```

!> 注意： 要启用输入（和参数）的自动验证，必须使用ValidationPipe。[了解更多有关验证](/5.0/techniques?id=验证)或者更具体的[管道](/5.0/pipes)。


## 变更（Mutations）

在 GraphQL 中，为了修改服务器端数据，我们使用了变更（[在这里阅读更多](http://graphql.cn/learn/queries/#mutations)） 。官方 Apollo 文档共享一个 upvotePost() 变更示例。该变更允许增加 votes 属性值。为了在 Nest 中创建等效变更，我们将使用 @Mutation() 装饰器。让我们 AuthorResolver 在上一节中扩展我们的用法（参见[解析图](/5.0/graphql?id=解析图)）。

```typescript
@Resolver('Author')
export class AuthorResolver {
  constructor(
    private readonly authorsService: AuthorsService,
    private readonly postsService: PostsService,
  ) {}

  @Query('author')
  async getAuthor(@Args('id') id: number) {
    return await this.authorsService.findOneById(id);
  }

  @Mutation()
  async upvotePost(@Args('postId') postId: number) {
    return await this.postsService.upvoteById({ id: postId });
  }

  @ResolveProperty('posts')
  async getPosts(@Parent() { id }) {
    return await this.postsService.findAll({ authorId: id });
  }
}
```
 请注意，我们假设业务逻辑已移至 PostsService（分别查询 post 和 incrementing votes 属性）。

### 类型定义

最后一步是将我们的变更添加到现有的类型定义中。


```typescript
type Author {
  id: Int!
  firstName: String
  lastName: String
  posts: [Post]
}

type Post {
  id: Int!
  title: String
  votes: Int
}

type Query {
  author(id: Int!): Author
}

type Mutation {
  upvotePost(postId: Int!): Post
}
```

该 `upvotePost(postId: Int!): Post` 变更现在可用！


## 订阅（Subscriptions）

订阅只是查询和变更的另一种 GraphQL 操作类型。它允许通过双向传输层创建实时订阅，主要通过 websockets 实现。[阅读更多]（https://www.apollographql.com/docs/graphql-subscriptions）(https://www.apollographql.com/docs/graphql-subscriptions/) 。

以下是 `commentAdded` 订阅示例，可直接从官方 [Apollo](https://www.apollographql.com/docs/graphql-subscriptions/subscriptions-to-schema.html) 文档复制和粘贴：

```typescript
Subscription: {
  commentAdded: {
    subscribe: () => pubSub.asyncIterator('commentAdded')
  }
}
```

?> `pubsub` 是一个 `PubSub` 类的实例。在[这里](https://www.apollographql.com/docs/graphql-subscriptions/setup.html)阅读更多

为了以Nest方式创建等效订阅，我们将使用 `@Subscription()` 装饰器。让我们扩展 `AuthorResolver` 在解析器映射部分中的使用。

```typescript
const pubSub = new PubSub();

@Resolver('Author')
export class AuthorResolver {
  constructor(
    private readonly authorsService: AuthorsService,
    private readonly postsService: PostsService,
  ) {}

  @Query('author')
  async getAuthor(@Args('id') id: number) {
    return await this.authorsService.findOneById(id);
  }

  @ResolveProperty('posts')
  async getPosts(@Parent() author) {
    const { id } = author;
    return await this.postsService.findAll({ authorId: id });
  }

  @Subscription()
  commentAdded() {
    return {
      subscribe: () => pubSub.asyncIterator('commentAdded'),
    };
  }
}
```

我们在这里使用了一个本地 `PubSub` 实例。相反, 我们应该将 `PubSub` 定义为一个组件, 通过构造函数 (使用 `@Inject ()` 装饰器) 注入它, 并在整个应用程序中重用它。[您可以在此了解有关嵌套自定义组件的更多信息](/5.0/fundamentals?id=自定义providercustomer-provider)。

### Module

为了启用订阅，我们必须将 `installSubscriptionHandlers` 属性设置为 `true` 。

```typescript
GraphQLModule.forRoot({
  typePaths: ['./**/*.graphql'],
  installSubscriptionHandlers: true,
})
```

要自定义订阅服务器（例如，更改端口），您可以使用 `subscriptions` 属性（阅读[更多](https://www.apollographql.com/docs/apollo-server/v2/api/apollo-server.html#constructor-options-lt-ApolloServer-gt)）。

### 类型定义

最后一步是更新类型定义文件。

```typescript
type Author {
  id: Int!
  firstName: String
  lastName: String
  posts: [Post]
}

type Post {
  id: Int!
  title: String
  votes: Int
}

type Query {
  author(id: Int!): Author
}

type Comment {
  id: String
  content: String
}

type Subscription {
  commentAdded(repoFullName: String!): Comment
}
```

做得好。我们创建了一个commentAdded(repoFullName: String!): Comment订阅。您可以在[此处](https://github.com/nestjs/nest/blob/master/sample/12-graphql-apollo)找到完整的示例实现。

## 标量

为了定义一个自定义标量（在[这里](http://graphql.cn/learn/schema/#scalar-types)阅读更多关于标量的信息），我们必须创建一个类型定义和一个专用的解析器。在这里（如在官方文档中），我们将采取 `graphql-type-json` 包用于演示目的。这个npm包定义了一个`JSON`GraphQL标量类型。首先，让我们安装包：

```bash
$ npm i --save graphql-type-json
```

然后，我们必须将自定义解析器传递给 `forRoot()` 函数：

```typescript
import * as GraphQLJSON from 'graphql-type-json';

@Module({
  imports: [
    GraphQLModule.forRoot({
      typePaths: ['./**/*.graphql'],
      resolvers: { JSON: GraphQLJSON },
    }),
  ],
})
export class ApplicationModule {}
```
现在, 我们可以在类型定义中使用 `JSON` 标量:

```typescript
scalar JSON

type Foo {
  field: JSON
}
```

### 类

定义标量类型的另一种形式是创建一个简单的类。假设我们想用 `Date`类型增强我们的模式。

```typescript
import { Scalar } from '@nestjs/graphql';
import { Kind } from 'graphql';

@Scalar('Date')
export class DateScalar {
  description = 'Date custom scalar type';

  parseValue(value) {
    return new Date(value); // value from the client
  }

  serialize(value) {
    return value.getTime(); // value sent to the client
  }

  parseLiteral(ast) {
    if (ast.kind === Kind.INT) {
      return parseInt(ast.value, 10); // ast value is always in string format
    }
    return null;
  }
}
```
之后，我们需要注册 `DateScalar` 为提供者。

```
@Module({
  providers: [DateScalar],
})
export class CommonModule {}
```
现在我们可以在 `Date` 类型定义中使用标量。


## 工具

在GraphQL世界中，很多文章抱怨如何处理诸如身份验证或操作的副作用之类的东西。我们应该把它放在业务逻辑中吗？我们是否应该使用更高阶的函数来增强查询和变更，例如，使用授权逻辑？或者也许使用[模式指令](https://www.apollographql.com/docs/apollo-server/v2/features/directives.html)。无论如何，没有一个答案。

Nest生态系统正试图利用看守器和拦截器等现有功能帮助解决这个问题。它们背后的想法是减少冗余，并为您提供有助于创建结构良好，可读且一致的应用程序的工具。

### 概述

您可以以与简单的REST应用程序相同的方式使用看守器、拦截器和管道。此外，您还可以通过利用自定义 decorator 特性轻松地创建自己的 decorator。他们都一样。让我们看看下面的代码:

```typescript
@Query('author')
@UseGuards(AuthGuard)
async getAuthor(@Args('id', ParseIntPipe) id: number) {
  return await this.authorsService.findOneById(id);
}
```

正如您所看到的，GraphQL在看守器和管道方面都能很好地工作。因此，您可以将身份验证逻辑移至看守器，甚至可以复用与 REST 应用程序相同的看守器类。拦截器的工作方式完全相同：

```typescript
@Mutation()
@UseInterceptors(EventsInterceptor)
async upvotePost(@Args('postId') postId: number) {
  return await this.postsService.upvoteById({ id: postId });
}
```
### 执行上下文

但是，ExecutionContext 看守器和拦截器所接收的情况有所不同。GraphQL 解析器有一个单独的参数集，分别为 root，args，context，和 info。因此，我们需要将 ExecutionContext 转换为 GqlExecutionContext，这非常简单。

```typescript
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { GqlExecutionContext } from '@nestjs/graphql';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const ctx = GqlExecutionContext.create(context);
    return true;
  }
}
```

GqlExecutionContext 为每个参数公开相应的函数，比如 getArgs()，getContext()等等。现在，我们可以毫不费力地获取特定于当前处理的请求的每个参数。


### 自定义装饰器

如前所述，自定义装饰器功能也可以像 GraphQL 解析器一样工作。但是，Factory 函数采用一组参数而不是 request 对象。

```
export const User = createParamDecorator(
  (data, [root, args, ctx, info]) => ctx.user,
);
```

然后：

```typescript
@Mutation()
async upvotePost(
  @User() user: UserEntity,
  @Args('postId') postId: number,
) {}
```

 ### 译者署名

| 用户名 | 头像 | 职能 | 签名 |
|---|---|---|---|
| [@zuohuadong](https://www.zhihu.com/people/dongcang)  | <img class="avatar-66 rm-style" src="https://pic.downk.cc/item/5f4cafe7160a154a67c4047b.jpg">  |  翻译  | 专注于 caddy 和 nest，[@zuohuadong](https://github.com/zuohuadong/) at Github  |
| [@Drixn](https://drixn.com/)  | <img class="avatar-66 rm-style" src="https://cdn.drixn.com/img/src/avatar1.png">  |  翻译  | 专注于 nginx 和 C++，[@Drixn](https://drixn.com/) |
