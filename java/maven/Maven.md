# Maven



## 多模块项目

单模块 -> 多模块，重构

- 调整主（父）工程类型（`<packaging>`）
- 创建子模块工程（`<module>`）
  - 模型层：model
  - 持久层：persistence
  - 表示层：web



## dependencies与dependencyManagement的区别

- 父项目中的dependencies中定义的所有依赖，在子项目中都会直接继承
- 在父项目中的dependencyManagement中定义的所有依赖，子项目并不会继承，我们还要在子项目中引入我们需要的依赖，才能进行使用。此时我们在子项目中不用设置版本。

