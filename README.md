## koa session store in mongodb

koa实现session保存mongodb案例

由于此模块依赖 koa-session, 首先安装 koa-session

> npm install koa-session

在启动文件中加载模块
```
const session = require('koa-session');
const SessionStore = require('./sessionStore'); //假设文件sessionStore.js 文件保存在 core的根目录
```
配置session

```
app.keys = ['some secret hurr'];
// session 配置信息
const CONFIG = {
    key: 'koa:sess',
    maxAge: 86400000,
    overwrite: true, 
    httpOnly: true, 
    signed: true, 
    rolling: false,
};
```
```
// 以中间件的方式使用session 
app.use(session(CONFIG, app));
```

设置session保存mongodb中

以上的配置并没有吧session信息保存在mongodb中，所以继续...

在koa-session的说明中， 如果要吧session信息保存在数据库中， 可以自己实现： 

`https://github.com/koajs/session#external-session-stores`

在CONFIG中增加store参数：

```
const CONFIG = {
        key: 'koa:sess',
        maxAge: 86400000,
        overwrite: true, 
        httpOnly: true, 
        signed: true, 
        rolling: false,
        store: new SessionStore({
            collection: 'navigation', //数据库集合
            connection: Mongoose,     // 数据库链接实例
            expires: 86400, // 默认时间为1天
            name: 'session' // 保存session的表名称
        })
    };
```
测试并使用session

你可以在你想要用session的地方获取到session

demo:
```
app.use(async (ctx, next) => {
    // 获取session对象
    const session = ctx.session;

    // 给session赋值
    session.userInfo = {
        name:'anziguoer',
        email:'anziguoer@163.com',
        age : 28
    }
    next();
})
```
接下来你查看mongodb数据库中是否以保存了你想要的值
sessionStore.js 文件源码, 你可以复制代码到你项目的任何目录， 只要保存引用正确即可

```
const schema = {
    _id: String,
    data: Object,
    updatedAt: {
		default: new Date(),
		expires: 86400, // 1 day
		type: Date
    }
};
export default class MongooseStore {
	constructor ({
		collection = 'sessions',
		connection = null,
		expires = 86400,
		name = 'Session'
		} = {}) {
			if (!connection) {
			throw new Error('params connection is not collection');
			}
		const updatedAt = { ...schema.updatedAt, expires };
		const { Mongo, Schema } = connection;
		this.session = Mongo.model(name, new Schema({ ...schema, updatedAt }));
	}
	
	async destroy (id) {
		const { session } = this;
		return session.remove({ _id: id });
	}
	
	async get (id) {
		const { session } = this;
		const { data } = await session.findById(id);
		return data;
	}
	
	async set (id, data, maxAge, { changed, rolling }) {
		if (changed || rolling) {
		const { session } = this;
		const record = { _id: id, data, updatedAt: new Date() };
		await session.findByIdAndUpdate(id, record, { upsert: true, safe: true });
		}
		return data;
	}
	
	static create (opts) {
		return new MongooseStore(opts);
	}
}
      
```
