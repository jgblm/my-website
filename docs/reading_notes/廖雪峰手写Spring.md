# 廖雪峰手写Spring

## 1. 实现IOC容器

IOC容器的功能：通过@ComponentScan扫描package下的所有@Configuration和@Component，根据Component的依赖关系，实例化和注入Component，放入ApplicationContext容器中。

1. 扫描获取所有的类 **ResourceResolver**
2. 配置属性获取与解析  **PropertyResolver**
3. 为了介入Bean的整个创建过程，并不直接创建Bean实例，而是先根据配置创建**BeanDefination**.
4. 根据BeanDefination创建**Bean**。对于存在强依赖的Bean，需要递归创建依赖的bean。对于弱依赖的bean，创建完成后再进行set。
5. 初始化Bean，调用initMethod。
6. **BeanPostProcessor**用来替换bean，比如代理类。为了保证被注入时使用的是代理类，注入时使用的是原始类，BeanPostProcessor需要提供一个方法获取原始的bean。

