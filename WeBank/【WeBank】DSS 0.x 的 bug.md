# 【WeBank】DSS 0.x 的 Q&A

1. 登录页面没有新建用户

2. azkaban 和 dss 用户同步.

   例如: DSS 登录用户为 hadoop，则需要同时在 dss-server 的 token.properties 和 azkaban的azkaban-users.xml 里面添加该用户

   在 dss-server 的 token.properties 添加用户

   ```shell
   sudo vi dss/dss-server/conf/token.properties
   hadoop=hadoop
   ```

   在azkaban的azkaban-users.xml里面添加用户

   ```xml
   sudo vi azkaban/conf/token.xml
   <user username="hadoop" password="hadoop" />
   ```

   