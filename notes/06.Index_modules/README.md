## Index modules

索引模块控制着模块的各个方面。

### Index Settings

配置可能是：

- static

  配置只能在索引创建时设置或者关闭的状态下修改；

- dynamic

  可以随时使用 `update-index-settings` 接口修改；

**对一个关闭着的索引进行静态或动态配置修改可能导致错误设置，甚至需要删除或者重建这个索引来修复。**

#### Static index settings

下面是不与任何特定索引模块关联的静态索引设置列表；

#### Dynamic index settings

