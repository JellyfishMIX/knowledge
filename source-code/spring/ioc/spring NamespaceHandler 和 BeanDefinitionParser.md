# spring NamespaceHandler 和 BeanDefinitionParser



## BeanDefinitionParser

解析相关节点，并注册BeanDefinition。

大致流程为解析 <context:property-placeholder location="classpath:module.properties" /> 节点，包装PropertySourcesPlaceholderConfigurer为BeanDefinition，将里面的属性装配到BeanDefinition中，并注册到BeanDefinitionMap。