## 工程结构规约

### 公共服务

1. 公共服务所有模块采用统一的group名字如com.chen.common
2. 按提供的能力类型做切分，如redis，http，mysql，log。采用多个模块发布，有利于依赖的管理，基础服务按需依赖公共服务而不是直接依赖全部的基础服务

### 基础服务（指实际提供http和rpc接口的服务）
1. 所有接口上必须有详细的注释说明这个使用接口的限制等

### GAV规则

### 包名规则(针对基础服务)

1. 所有的包名以com.公司名.项目名.模块名打头 例子：com.chen.game.user ，指明这是chen公司game项目的user模块
2. controller包下提供http接口，service包下提供rpc接口


### 