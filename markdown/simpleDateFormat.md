# SimpleDateFormat线程不安全 及 解决方案
---
## 一 SimpleDateFormat线程不安全原因

        // SimpleDateFormat 对象中包含calendar对象 用于format
        protected Calendar calendar;

        // Called from Format after creating a FieldDelegate
        private StringBuffer format(Date date, StringBuffer toAppendTo,
                                    FieldDelegate delegate) {
            // 每次format时 都会给calendar 对象赋值 此处是造成 static SimpleDateFormat 线程不安全的根本原因            
            // Convert input date to time field list
            calendar.setTime(date);
    
            boolean useDateFormatSymbols = useDateFormatSymbols();
    
            for (int i = 0; i < compiledPattern.length; ) {
                int tag = compiledPattern[i] >>> 8;
                int count = compiledPattern[i++] & 0xff;
                if (count == 255) {
                    count = compiledPattern[i++] << 16;
                    count |= compiledPattern[i++];
                }
    
                switch (tag) {
                case TAG_QUOTE_ASCII_CHAR:
                    toAppendTo.append((char)count);
                    break;
    
                case TAG_QUOTE_CHARS:
                    toAppendTo.append(compiledPattern, i, count);
                    i += count;
                    break;
    
                default:
                    subFormat(tag, count, delegate, toAppendTo, useDateFormatSymbols);
                    break;
                }
            }
            return toAppendTo;
        }
        
## 二 SimpleDateFormat线程不安全 解决方按
### 1.不使用 static SimpleDateFormat 每次使用都创建新的对象
问题：并发高时 创建了太多的SimpleDateFormat对象 占用堆内存 
 
### 2.使用 static SimpleDateFormat 加锁或同步代码块 解决线程安全问题
问题：并发高时 性能问题

### 3.使用ThreadLocal

        private static final String date_format = "yyyy-MM-dd HH:mm:ss";
        private static ThreadLocal<DateFormat> threadLocal = new ThreadLocal<>();
    
        public static DateFormat getDateFormat() {
            DateFormat dateFormat = threadLocal.get();
            if (null == dateFormat) {
                dateFormat = new SimpleDateFormat(date_format);
                threadLocal.set(dateFormat);
            }
            return dateFormat;
        }
    
        public static String formatDate(Date date) throws ParseException {
            return getDateFormat().format(date);
        }
    
        public static Date parse(String strDate) throws ParseException {
            return getDateFormat().parse(strDate);
        }
        
 ### 4.使用ThreadLocal
 (1)使用Apache commons 里线程安全的FastDateFormat, 可惜它只能对日期进行format, 不能对日期串进行解析<br>
 (2)使用Joda-Time类库来处理时间相关问题