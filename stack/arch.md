# 架构

## 软件架构

| 微服务类别 | 建议软件架构                   | 说明                                                 |
|-------|--------------------------|----------------------------------------------------|
| 业务系统  | 整洁架构（Clean Architecture） | 业务系统一般有MySQL、Redis、Kubernetes等外部依赖； 可适当引入领域驱动设计思想； |
| 基础服务  | 灵活                       | 业务无关，较少或没有外部依赖                                     |
| 其他服务  | 灵活	                      | 非重要服务，可使用高效开发框架                                    |

## 领域驱动设计和整洁架构

|       | 领域驱动设计（DDD）                                                                                                           | 整洁架构（Clean Architecture）                                                | 函数式编程及面向接口编程                                       |
|-------|-----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------|----------------------------------------------------|
| 侧重点   | 提高代码可读性、提高开发效率	                                                                                                       | 易于测试、维护及重构                                                              | /                                                  | 
| 关注点   | 业务逻辑，领域模型                                                                                                             | 	应用程序内部结构，依赖反转	                                                         | /                                                  | 
| 优点    | 提高代码可读性、可维护性	                                                                                                         | 提高代码可测试性、可重构性                                                           | /                                                  | 
| 缺点    | 学习曲线高、不适用于所有项目                                                                                                        | 忽略内部结构、不够灵活	                                                            | 只是编程方式，大型项目还需结合程序架构                                | 
| 适用场景  | 业务逻辑复杂，业务领域易于理解	                                                                                                      | 需要提高测试覆盖率、可重构性的项目                                                       | 所有场景                                               | 
| 适用案例  | 在金融领域，DDD 可以用来模型化银行账户、支票、信用卡等概念，从而提高代码的可读性和可维护性	                                                                      | 构建分布式系统，例如微服务架构，从而使得每个微服务都能够独立测试和部署	                                    | 所有项目                                               | 
| 不适用场景 | 业务逻辑简单，业务领域难以理解	                                                                                                      | 内部结构非常复杂，无法通过 Clean Architecture 来简化，反而会复杂化	                            | 无                                                  | 
| 建议    | 边缘计算相关项目，业务逻辑复杂度有限，甚至没有业务逻辑，不建议使用，但可借鉴一些设计思想。此外，即使有较多业务逻辑的场景，在没有领域专家，团队成员没有实战经验的情况下，也不建议使用，不合适的领域建模可能引起无法阅读或无法维护的项目。  | 边缘计算会涉及到一些业务逻辑的项目推荐使用，可提高代码可测试性，从而提高系统稳定性。边缘计算无业务逻辑项目，不推荐使用，比如网络配置程序等。	 | 基础库，无业务逻辑项目。大型项目需结合 Clean Architecture、 DDD 等程序架构。 | 

## 继承、组合及接口

|      | 继承                                                                        | 组合                                     | 接口                                          |
|------|---------------------------------------------------------------------------|----------------------------------------|---------------------------------------------|
| 用途   | 一个类可以从另一个类那里获取属性和方法                                                       | 	一种将对象组合成树形结构来表示“部分-整体”层次结构的方法         | 一种定义了某类对象的抽象方法的集合，可以让类实现特定的功能，而不必关心如何实现这些功能 |
| 优点   | 派生类可从直接使用基类的属性和方法，提高代码复用性                                                 | 可以让程序更具灵活，更容易扩展                        | 不必关心类的具体实现，降低类与类直接的耦合性                      |
| 缺陷   | 导致类之间的耦合性增强，使得类之间的关系变得更加复杂；子类覆盖父类的定义，可能会导致意想不到的结果；单继承无法解决一些问题，而多继承会导致钻石问题 | 可能会使代码变得复杂（维护组合对象和叶子对象的层次结构，区分不同的处理流程） | 可能会增加实现类复杂度                                 |
| 使用场景 | Golang不支持继承，无法使用，更没必要去模拟。抽离出公共函数一样可以达到代码复用的效果，只是组织松散一些。                   | 常用	                                    | 常用                                          |