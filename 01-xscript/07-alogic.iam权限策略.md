> 用户进了团队，作为一名成员，承担对应的角色，赋予相应的权限。

> 角色是基于服务的 **权限策略的组合**。

> workspace  <--   **Role**   -->  policy [service + **resource** + **action** + **mode**]
>

<img src=".\pictures\image-20201222092954170.png" alt="image-20201222092954170" style="zoom:80%;" />

> **制定实体（如服务）的权限策略需要**：1. 动作列表 actionList  2. 资源列表 ResourceList 

![image-20201222093814577](.\pictures\image-20201222093814577.png)

>权限策略：某工作区某角色（who），对于某服务（at）中的某资源（on），具有某操作权限（do）

