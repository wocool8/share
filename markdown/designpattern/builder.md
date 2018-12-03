# Builder
---
### The algorithm for creating a complex object should be independent of the parts that make up the object and how they're assembled.The construction process must allow different representations for the object that's constructed.

![Builder](../../picture/designpattern/builder.png)
### Lombok的@Builder为例介绍
- 编译前实体类
    ```java
    @Builder
    public class User implements Serializable {
    
        private static final long serialVersionUID = 1L;
    
        private int userId;
        private int empId;
        private String account;
        private String password;
    }
    ```
- 编译后
    ```java
    public class User implements Serializable {
        private static final long serialVersionUID = 1L;
        private int userId;
        private int empId;
        private String account;
        private String password;
    
        User(int userId, int empId, String account, String password) {
            this.userId = userId;
            this.empId = empId;
            this.account = account;
            this.password = password;
        }
    
        public static User.UserBuilder builder() {
            return new User.UserBuilder();
        }
    
        public static class UserBuilder {
            private int userId;
            private int empId;
            private String account;
            private String password;
    
            UserBuilder() {
            }
    
            public User.UserBuilder userId(int userId) {
                this.userId = userId;
                return this;
            }
    
            public User.UserBuilder empId(int empId) {
                this.empId = empId;
                return this;
            }
    
            public User.UserBuilder account(String account) {
                this.account = account;
                return this;
            }
    
            public User.UserBuilder password(String password) {
                this.password = password;
                return this;
            }

            public User build() {
                return new User(this.userId, this.empId, this.account, this.password);
            }
        }
    }
    ```
- 使用
    ```java
    User mumu = User.builder()
         .account("mumu")
         .password("123456")
         .userId(123)
         .empId(456)
         .build();
    ```