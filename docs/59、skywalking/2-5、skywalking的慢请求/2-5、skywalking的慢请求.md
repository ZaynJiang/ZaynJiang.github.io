## 1. 背景

在skywalking中，慢的sql请求是如何记录的呢

mysql-8.x-plugin 插件中会拦截 preparedStatement.execute() 方法创建 Database 类型的 ExitSpan，并在  execute() 方法调用完成之后结束 ExitSpan



SpanListener.parseExit() 方法还会针对 Database 类型的 ExitSpan 进行特殊处理，该处理主要用于统计慢查询。这里的慢查询统计不仅是 DB 的慢查询，还包括其他常见的存储，例如：Redis、MongoDB 等等