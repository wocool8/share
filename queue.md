# 队列
---
## 一 异常介绍
![exception](./exception.PNG)
## 二 异常处理方式
### 2.1 throw
    
        public static Statement parse(String sql) throws JSQLParserException {
            CCJSqlParser parser = new CCJSqlParser(new StringProvider(sql));
            try {
                return parser.Statement();
            } catch (Exception var3) {
                throw new JSQLParserException(var3);
            }
        }
    
### 2.2 try catch
    
        try {
            saveReadWriteRecord(dmlRecord);
            saveIndexRecord(dmlRecord);
        } catch (JSQLParserException e) {
            log.error("dmlRecord:{},Sql Parser Exception", JSON.toJSONString(dmlRecord), e);
            saveDmlRecordHandleResult(dmlRecord, DmlRecordStatusEnum.FAIL_PARSER_SQL.getCode());
        } catch (Exception e) {
            log.error("dmlRecord:{},save indexRecord Exception", JSON.toJSONString(dmlRecord), e);
            saveDmlRecordHandleResult(dmlRecord, DmlRecordStatusEnum.FAIL_SAVE_INDEX_RECORD.getCode());
        }
    
## 三 开源框架中的异常处理
### (1)事物xml/注解
### (2)使用自定义RuntimeException
### (3)国际化处理
### (4)性能损耗Throwable.fillInStackTrace
