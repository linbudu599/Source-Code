# Egg-GraphQL

## Egg 插件机制

要开始读之前当然要搞懂 Egg 的插件机制了, 官方文档讲的还是很清楚的, 插件!==中间件, 最明显的区别就是插件实际上就是独立的应用, 中间件则是附属于路由的. 插件包含 service/中间件/配置, 还可以再做扩展. 但是它没有自己独立的路由和控制器, 也没有 plugin.js, 只能声明对其他插件依赖, 而不能决定其他插件开启与否.

还有一点我觉得比较重要的, 当一个应用装载了多个插件, 实际上就可以被抽离为一个上层框架.

启用的话, 在`config/plugin.js`中声明其开启, 就会去依赖中找这个插件? 然后 egg 的注入机制会将其挂载到 app 上设定的命名空间, 如`app.mysql`这样.

(命名空间为插件`package.json`下的`eggPlugin.name`字段, `eggPlugin`下还有`dependencies`/`env`/`optionalDependencies`字段)

## 插件作用

- 扩展内置对象接口, 同在应用中扩展内置对象的方式, 即在ctx上进行扩展.
- 自定义的中间件实现
- 在应用启动前做一些操作

## Egg-GraphQL-Example

分析一下这个例子的实现

- 按照曾经的规范(现在的规范好像变了), 分为schema/resolver/connector这几个模块

- 各个文件夹的type/fragment都是互通的, 我的理解是Egg的约定式做了搜集整理这种

- 这个例子里还有个地方, 自定义标量(Scalar)

  ```graphql
  scalar Date
  ```

  ```js
  // resolver.js
  
  module.exports = {
    Date: require('./scalars/date'), // eslint-disable-line
  };
  
  //scalar/date.js
  const { GraphQLScalarType } = require('graphql');
  const { Kind } = require('graphql/language');
  
  module.exports = new GraphQLScalarType({
    name: 'Date',
    description: 'Date custom scalar type',
    parseValue(value) {
      return new Date(value);
    },
    serialize(value) {
      return value.getTime();
    },
    parseLiteral(ast) {
      if (ast.kind === Kind.INT) {
        return parseInt(ast.value, 10);
      }
      return null;
    },
  });
  ```

- connector, 在这里主要是去调用ORM取数据, 举例:

  ```js
  'use strict';
  
  const _ = require('lodash');
  
  class ItemConnector {
    constructor(ctx) {
      this.ctx = ctx;
      // 这里是在model中定义的
      this.proxy = ctx.app.model.Item;
    }
  
    async fetchByUserId(userID) {
      const tags = await this.proxy.findAll({
        where: {
          user_id: userID,
        },
      }).then(ts => ts.map(u => u.toJSON()));
      return tags;
    }
  }
  ```

- resolver, 这里主要是对应Query/Mutation, 将参数传入给resolver

- 猜想要做的事情:

  - 收集所有`schema.graphql`
  - 将connector挂载到ctx上
  - 将resolver和上面二者一一对应起来
  - 将`/graphql`的请求打到这个插件的`Apollo-Server-Koa`下?(通过中间件的机制拦截)



## 开始阅读源码

从`app.js`开始:

```js
'use strict';

module.exports = app => {
  require('./lib/load_schema')(app);
  require('./lib/load_connector')(app);
};
```

很明显是将app经过了两次处理, 比较符合上面的猜想即收集`schema`和`connector`

`load_schema.js`与`load_connector.js`, 最新版的代码和官方demo使用的版本不一样, 主要是添加了指令集支持等等

```js
// 收集schema/resolver等, 生成SDL, 挂载到app上, key值使用Symbol
const SYMBOL_SCHEMA = Symbol('Applicaton#schema');
const util = require('./util');

module.exports = app => {
  const basePath = path.join(app.baseDir, 'app/graphql');
  const types = util.walk(basePath, basePath);

  // 收集schema
  const schemas = [];
  // 收集resolver映射关系
  const resolverMap = {};
  const resolverFactories = [];
  const directiveMap = {};
  // 收集指令
  const schemaDirectivesProps = {};
  const { defaultEmptySchema = false } = app.config.graphql;
  const defaultSchema = `
    type Query 
    type Mutation 
  `;
  if (defaultEmptySchema) {
    schemas.push(defaultSchema);
  }
  types.forEach(type => {
    // schema.graphql文件
    const schemaFile = path.join(basePath, type, 'schema.graphql');
    /* istanbul ignore else */
    if (fs.existsSync(schemaFile)) {
      const schema = fs.readFileSync(schemaFile, {
        encoding: 'utf8',
      });
      schemas.push(schema);
    }

    // Load resolver
    const resolverFile = path.join(basePath, type, 'resolver.js');
    if (fs.existsSync(resolverFile)) {
      const resolver = require(resolverFile);
      if (_.isFunction(resolver)) {
        resolverFactories.push(resolver);
      } else if (_.isObject(resolver)) {
        _.merge(resolverMap, resolver);
      }
    }
	
    // 收集指令等, 这里跳过

  Object.defineProperty(app, 'schema', {
    get() {
      if (!this[SYMBOL_SCHEMA]) {
        // 交由工厂处理后 进行对应合并
        resolverFactories.forEach(resolverFactory => _.merge(resolverMap, resolverFactory(app)));

        this[SYMBOL_SCHEMA] = makeExecutableSchema({
          // 调用tool的方法 生成可执行范式
          typeDefs: schemas,
          resolvers: resolverMap,
        });
      }
      return this[SYMBOL_SCHEMA];
    },
  });
};
                
// load_connector.js
// 类似的  收集connector.js  并挂载到app上
const SYMBOL_CONNECTOR_CLASS = Symbol('Application#connectorClass');

module.exports = app => {
  const basePath = path.join(app.baseDir, 'app/graphql');
  const types = util.walk(basePath);
	
  // app.connectorClass后面会在extend中用到, 对ctx进行增强
  Object.defineProperty(app, 'connectorClass', {
    get() {
      if (!this[SYMBOL_CONNECTOR_CLASS]) {
        const classes = new Map();

        types.forEach(type => {

          const connectorFile = path.join(basePath, type, 'connector.js');
          /* istanbul ignore else */
          if (fs.existsSync(connectorFile)) {
            const Connector = require(connectorFile);
            classes.set(path.basename(type), Connector);
          }
        });

        this[SYMBOL_CONNECTOR_CLASS] = classes;
      }
      return this[SYMBOL_CONNECTOR_CLASS];
    },
  });
};
```

这样已经完成相关文件收集处理, 并挂载到app上.

在上面的使用可以看到, 在resolver里我们是调用connector的, 并且是通过`ctx.connector`的方式, 要实现这个需要对框架的ctx属性扩展, 即`extend/context.js`

官方推荐使用Symbol+Getter的方式

```js
'use strict';

const SYMBOL_CONNECTOR = Symbol('connector');

module.exports = {

  get connector() {
    /* istanbul ignore else */
    if (!this[SYMBOL_CONNECTOR]) {
      const connectors = {};
      // 使用前面挂载好的connector
      for (const [ type, Class ] of this.app.connectorClass) {
        connectors[type] = new Class(this);
      }
      this[SYMBOL_CONNECTOR] = connectors;
    }
    return this[SYMBOL_CONNECTOR];
  },

  get graphql() {
    return this.service.graphql;
  },
};
```

在使用demo中可以看到connector是类, 于是这里采用实例化的方式, 挂载到connector上, 于是就可以像`ctx.connector.xxx`的方式来使用

**connector** **resolver** **schema**的连结是通过GraphQLJS官方提供的工具包, 不是在插件层面做的实现.

其中`graphql`的访问器主要是在`service/graphql.js`做的实现, 新版的代码比旧版在可读性和简洁上做了不少优化.

注意这个文件做了什么, 主要是每一次你发送来的请求(`/graphql`下的)会被交由这里进行处理, (我的理解是mutation实际上也会被交由query处理)

```js
module.exports = app => {
  class GraphqlService extends app.Service {

    async query(requestString) {
      let result = {};
      const ctx = this.ctx;

      try {
        // 解析请求
        const params = JSON.parse(requestString);
        // 语句 参数
        // operationName操作名称, 即query/mutation/subscription
        const { query, variables, operationName } = params;
          
        const documentAST = gql`${query}`;
        // 这里我觉得挺神奇的  Koa(Egg)的ctx竟然能给它用, 虽然说二者都是挂载属性的方式
        const context = ctx;
        // 也是前面挂载好的 也就是经过tool处理过的可执行schema
        const schema = this.app.schema;

        result = await execute(
          schema,
          documentAST,
          null,
          context,
          variables,
          operationName
        );

        if (result && result.errors) {
          result.errors = result.errors.map(formatError);
        }
      } catch (e) {
        this.logger.error(e);

        result = {
          data: {},
          errors: [ e ],
        };
      }
      return result;
    }
  }
  return GraphqlService;
};
```

这里主要就是执行处理每一次GraphQL查询, 主要逻辑也是由GraphQL JS实现的



然后就是处理GraphQL请求了, (实际上和上面顺序反了, 应该是先接到请求然后处理请求, 不过这样更好理解)



这里大致是我猜测的思路: 即在中间件(路由)中捕获请求处理

`middleware/graphql.js`, 这也是为什么使用的时候还需要在配置里加上中间件的配置项

这里有一些兼容性问题, 详情可以查看官方源码的注释

```js
const { graphqlKoa } = require('apollo-server-koa/dist/koaApollo');
const { resolveGraphiQLString } = require('apollo-server-module-graphiql');

// 解析GraphIQL, 返回IQL界面, 注意GraphIQL是官方提供的, GraphQL Playground是Apollo自己的实现, 更好看更灵活更流畅一些
function graphiqlKoa(options) {
  return ctx => {
    const query = ctx.request.query;
    return resolveGraphiQLString(query, options, ctx)
      .then(graphiqlString => {
        ctx.set('Content-Type', 'text/html');
        ctx.body = graphiqlString;
      });
  };
}

module.exports = (_, app) => {
  // 收集配置及路由
  const options = app.config.graphql;
  const graphQLRouter = options.router;
  let graphiql = true;

  if (options.graphiql === false) {
    graphiql = false;
  }

  return async (ctx, next) => {
    /* istanbul ignore else */
    if (ctx.path === graphQLRouter) {
      const {
        onPreGraphiQL,
        onPreGraphQL,
        apolloServerOptions,
      } = options;
      // 只有启用iql才可启用onPreIQL
      if (ctx.request.accepts([ 'json', 'html' ]) === 'html' && graphiql) {
        if (onPreGraphiQL) {
          await onPreGraphiQL(ctx);
        }
        return graphiqlKoa({
          endpointURL: graphQLRouter,
        })(ctx);
      }
      if (onPreGraphQL) {
        await onPreGraphQL(ctx);
      }
      const serverOptions = Object.assign(
        {},
        apolloServerOptions,
        {
          schema: app.schema,
          context: ctx,
        }
      );
      return graphqlKoa(serverOptions)(ctx);
    }
    await next();
  };
};
// onPre*2 处理!
// 这里还支持了其他的apollo-server选项, 当然实现方式就是直接传给它...
```



大致思路即是如此, 在约定地目录下收集schema文件与connector&resolver, 并交由tool生成可执行schema, 挂载到`app.schema/app.connectorClass`下, 并按照文件夹名进行了配置如`app.connector.user`. 然后在extend中对context进行扩展, 包括`connector`与`graphql`, 注意这里是`ctx.connector.user`的底层实现. 而在`service/graphql.js`中, 主要负责对每一次请求进行解析处理与错误信息格式化(formatError). 而能接收到请求则要靠`middleware/graphql.js`来进行拦截, 以及GraphIQL的生成(html字符串), 两处onPre处理,  使用配置选项开启Apollo-Server-Koa这几个步骤. 我个人理解在启动后其请求就交给了service处理.