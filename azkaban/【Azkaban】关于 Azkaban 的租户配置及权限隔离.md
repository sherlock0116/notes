# 【Azkaban】关于 Azkaban 的租户配置及权限隔离

如有疑问, 详情参考[官网](https://azkaban.readthedocs.io/en/latest/userManager.html#xmlusermanager)

## 一、UserManager(用户管理)

启动 Azkaban 时，您可能会注意到登录页面。Azkaban 让您在使用前先进行身份验证。这可以防止看到或执行您不应该看到或触摸的工作流程。

我们还使用经过身份验证的用户进行审计。每当项目文件更改、修改、计划等时，我们通常想知道哪个用户执行了该操作。

**用户登录页面：**

![](https://azkaban.readthedocs.io/en/latest/_images/login.png)

### 1.1 XmlUserManager

XmlUserManager 是 Azkaban 内置的默认 UserManager。要显式设置配置 XmlUserManager 的参数，可以在 `azkaban.properties` 文件中设置以下参数 。

| Parameter             | Default                     |
| --------------------- | --------------------------- |
| user.manager.class    | azkaban.user.XmlUserManager |
| user.manager.xml.file | azkaban-users.xml           |

另一个需要修改的`azkaban-users.xml` 文件是文件。XmlUserManager 将在启动期间解析用户 xml 文件以设置用户。

一切都必须包含在一个`<azkaban-users>`标签中。

```xml
<azkaban-users>
    ...
</azkaban-users>
```

### 1.2 Users

要添加用户，请添加`<user>`标签。：

```xml
<azkaban-users>
  <user username="myusername" password="mypassword" roles="a" groups="mygroup" / >
  <user username="myusername2" password="mypassword2" roles="a, b" groups="ga, gb" / >
   ...
 </azkaban-users>
```

| Attributes    | Values                                                       | Required? |
| ------------- | ------------------------------------------------------------ | --------- |
| username      | The login username.                                          | yes       |
| password      | The login password.                                          | yes       |
| roles/角色    | Comma delimited list of roles that this user has.            | no        |
| groups/所属组 | Comma delimited list of groups that the users belongs to.    | no        |
| proxy/代理    | Comma delimited list of proxy users that this users can give to a project | no        |

### 1.3 Groups(所属组)

要添加用户，请添加`<group>`标签。：

```xml
<azkaban-users>
  <user username="a" ... groups="groupa" / >
  ...
  <group name="groupa" roles="myrole" / >
  ...
</azkaban-users>
```

在前面的示例中，用户 'a' 在组 'groupa' 中。用户“a”也将具有“myrole”角色。普通用户不能向项目添加组权限，除非他们是该组的成员。

以下是您可以分配的一些组属性。

| Attributes | Values                                            | Required? |
| ---------- | ------------------------------------------------- | --------- |
| name       | The group name                                    | yes       |
| roles      | Comma delimited list of roles that this user has. | no        |

### 1.4 Roles(用户权限)

角色的不同之处在于它为 Azkaban 中的用户分配全局权限。您可以使用`<roles>`标签设置角色。：

```xml
<azkaban-users>
  <user username="a" ... groups="groupa" roles="readall" / >
  <user username="b" ... / >
  ...
  <group name="groupa" roles="admin" / >
  ...
  <role name="admin" permissions="ADMIN" / >
  <role name="readall" permissions="READ" / >
</azkaban-users>
```

在上面的示例中，用户“a”具有角色“readall”，该角色被定义为具有 READ 权限。这意味着用户“a”对所有项目和执行具有全局读取访问权限。

用户 'a' 也在 'groupa' 中，它的角色是 ADMIN。这当然是多余的，但用户 'a' 也被授予所有项目的 ADMIN 角色。

以下是您可以分配的一些组属性。

| Attributes  | Values                                               | Required? |
| ----------- | ---------------------------------------------------- | --------- |
| name        | The group name                                       | yes       |
| permissions | Comma delimited list global permissions for the role | yes       |

可能的角色权限如下：

| Permissions    | Values                                                       |
| -------------- | ------------------------------------------------------------ |
| ADMIN          | Grants all access to everything in Azkaban.(授予对 Azkaban 中所有内容的所有访问权限。) |
| READ           | Gives users read only access to every project and their logs(为用户提供对每个项目及其日志的只读访问权限) |
| WRITE          | Allows users to upload files, change job properties or remove any project(允许用户上传文件、更改作业属性或删除任何项目) |
| EXECUTE        | Allows users to trigger the execution of any flow(允许用户触发任何流程的执行) |
| SCHEDULE       | Users can add or remove schedules for any flows(用户可以为任何流添加或删除计划) |
| CREATEPROJECTS | Allows users to create new projects if project creation is locked down(如果项目创建被锁定，则允许用户创建新项目) |

## 二、Custom User Manager

尽管 XmlUserManager 很容易上手，但您可能希望与已建立的目录系统（例如 LDAP）集成。

实现自定义 UserManager 应该是相当直接的。UserManager 是一个 java 接口。只需要实现几个方法：

```java
public interface UserManager {
    public User getUser(String username, String password) throws UserManagerException;
    public boolean validateUser(String username);
    public boolean validateGroup(String group);
    public Role getRole(String roleName);
    public boolean validateProxyUser(String proxyUser, User realUser);
}
```

构造函数应该接受一个 `azkaban.utils.Props ` 对象。`azkaban.properties` 的内容将可用于 UserManager 进行配置。

将您的新自定义 UserManager 打包成一个 jar 并将其放入 `./extlib`目录或插件目录（即 `./plugins/ldap/linkedin-ldap.jar`）。

将 `azkaban.properties` 配置更改为指向自定义 UserManager。`azkaban.properties`如果您的自定义用户管理器需要，可以添加其他参数。

| Parameter            | Default                          |
| -------------------- | -------------------------------- |
| `user.manager.class` | `azkaban.user.CustomUserManager` |

