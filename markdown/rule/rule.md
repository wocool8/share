# 规则引擎
---
## 一 QLExpress([文档连接](https://github.com/alibaba/QLExpress))
QLExpress是一个由阿里开源的轻量级的类java语法规则引擎，作为一个嵌入式规则引擎在业务系统中使用。让业务规则定义简便而不失灵活。让业务人员就可以定义业务规则。支持标准的JAVA语法，还可以支持自定义操作符号、操作符号重载、函数定义、宏定义、数据延迟加载等
![QLExpress](../../picture/rule/QLExpress.PNG)
## 二 Drools
Drools是最活跃的开源规则引擎。基于Drools的XML框架 + Java/Groovy/Python嵌入语言
![规则匹配](../../picture/rule/partternMatcher.PNG)
### 2.1 语言规则
    rule “name”
        attributes ---->属性
        when
            LHS ---->条件
        then
            RHS	---->结果
    end
### 2.2 and & or & Object
    rule "infixAnd"
        when
          ( $c1 : Customer ( country=="GB") and  PrivateAccount(  owner==$c1))
                or
           ( $c1 : Customer (country=="US") and PrivateAccount(  owner==$c1))
        then
            showResult.showText("Person lives in GB or US");
    end
  