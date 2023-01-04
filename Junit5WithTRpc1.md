# Junit5 With Spring&TRpc介绍
## Junit5与Junit4对比
| Junit5 | Junit4 |
|---|---|
|org.junit.jupiter.api.@Test|org.junit.@Test|
|Assertions|Assert|
|@Test(expected = Exception.class)|Assertions.assertThrows(Exception.class, () -> {})|
|@Test(timeout = 1)|Assertions.assertTimeout(Duration.ofMillis(1), () ->{}|
|@BeforeEach|@Before|
|@AfterEach|@After|
|@BeforeAll|@BeforeClass |
|@AfterAll|@AfterClass|
|@Disabled|@Ignore|
|support lambda||
|@ExtendWith()|@RunWith()|
|BeforeAllCallback, AfterAllCallback, ParameterResolver,BeforeEachCallback, AfterEachCallback, BeforeTestExecutionCallback, AfterTestExecutionCallback,TestExecutionExceptionHandler|TestRule|
|@Tag,@Tags|@Category|
|@DisplayName||
|不能有测试公共方法|可以有公共测试方法|
|Junit 5由3个子项目组成，即JUnit Platform，JUnit Jupiter和JUnit Vintage。**JUnit Platform**定义了TestEngine用于开发在平台上运行的新测试框架的API。**JUnit Jupiter**具有所有新的junit注释和TestEngine实现，以运行使用这些注释编写的测试。**JUnit Vintage**支持在JUnit 5平台上运行JUnit 3和JUnit 4编写的测试。|JUnit 4将所有内容捆绑到单个jar文件中|


## Junit5生命周期控制
### Extension
#### 原生Extension
```java
@ExtensionWith(LogExtension .class,AExtension.class,BExtension.class...)
```
```java
package com.fit.fx.test.starter.extension;

import java.lang.reflect.Method;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.extension.AfterEachCallback;
import org.junit.jupiter.api.extension.BeforeEachCallback;
import org.junit.jupiter.api.extension.ExtensionContext;
import org.junit.jupiter.api.extension.ExtensionContext.Namespace;
import org.junit.jupiter.api.extension.ExtensionContext.Store;

@Slf4j
public class LogExtension implements BeforeEachCallback, AfterEachCallback {

    private static final Namespace CURRENT_NS = Namespace.create(LogExtension.class);

    @Override
    public void afterEach(ExtensionContext context) throws Exception {
        final Store store = context.getRoot().getStore(CURRENT_NS);
        final Method testMethod = context.getRequiredTestMethod();
        log.info("test {} cost {}", testMethod.getName(), (System.currentTimeMillis() - (Long) store.get(testMethod)));

    }

    @Override
    public void beforeEach(ExtensionContext context) throws Exception {
        final Method testMethod = context.getRequiredTestMethod();
        log.info("test {} started", testMethod.getName());
        final Store store = context.getRoot().getStore(CURRENT_NS);
        store.put(testMethod, System.currentTimeMillis());
    }
}
```

#### SpringBased Extension
基于Spring上下文，可以获取Bean或者properties

```java
@ExtensionWith({DirtyResourceCleanUpExtension .class,SpringExtension.class})
```
```java
package com.fit.fx.test.starter.extension;

import org.junit.jupiter.api.extension.ExtensionContext;
import org.junit.jupiter.api.extension.ExtensionContext.Namespace;
import org.junit.jupiter.api.extension.ExtensionContext.Store;
import org.springframework.test.context.TestContextManager;
import org.springframework.test.context.junit.jupiter.SpringExtension;
import org.springframework.util.Assert;

public abstract class AbstractSpringExtension {

    //复用SpringExtension命名空间
    private static final Namespace TEST_CONTEXT_MANAGER_NAMESPACE = Namespace.create(SpringExtension.class);

    protected static TestContextManager getTestContextManager(ExtensionContext context) {
        Assert.notNull(context, "ExtensionContext must not be null");
        Class<?> testClass = context.getRequiredTestClass();
        Store store = getStore(context);
        return store.getOrComputeIfAbsent(testClass, TestContextManager::new, TestContextManager.class);
    }

    protected static Store getStore(ExtensionContext context) {
        return context.getRoot().getStore(TEST_CONTEXT_MANAGER_NAMESPACE);
    }

}

```
```java
package com.fit.fx.test.starter.extension;

import com.fit.fx.test.starter.DirtyResource;
import java.util.Collection;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.extension.AfterEachCallback;
import org.junit.jupiter.api.extension.ExtensionContext;
import org.springframework.test.context.TestContext;

@Slf4j
public class DirtyResourceCleanUpExtension extends AbstractSpringExtension implements AfterEachCallback {

    @Override
    public void afterEach(ExtensionContext context) throws Exception {
        final TestContext testContext = getTestContextManager(context).getTestContext();
        Collection<DirtyResource> dirtyResources = testContext.getApplicationContext()
                .getBeansOfType(DirtyResource.class)
                .values();
        for (DirtyResource dirtyResource : dirtyResources) {
            dirtyResource.cleanUp(testContext);
        }
    }
}
```
###TestExecutionListener
Spring提供的测试生命周期控制，本质是由SpringExtension触发注解中的Listener实现的不同生命周期的方法，通过TestContextManager管理Listener集合和触发不同生命周期方法，和SpringBased Extension比为比较方便控制顺序，并且不需要根据store中获取TestContextManager再获取TestContext(SpringExtension做了这件事)

```java
@TestExecutionListeners(SqlScriptsTestExecutionListener.class)
```
```java
package org.springframework.boot.test.mock.mockito;

import java.util.Arrays;
import java.util.HashSet;
import java.util.Set;

import org.mockito.Mockito;

import org.springframework.beans.factory.NoSuchBeanDefinitionException;
import org.springframework.beans.factory.config.BeanDefinition;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.core.Ordered;
import org.springframework.test.context.TestContext;
import org.springframework.test.context.TestExecutionListener;
import org.springframework.test.context.support.AbstractTestExecutionListener;
import org.springframework.util.ClassUtils;

/**
 * {@link TestExecutionListener} to reset any mock beans that have been marked with a
 * {@link MockReset}. Typically used alongside {@link MockitoTestExecutionListener}.
 *
 * @author Phillip Webb
 * @since 1.4.0
 * @see MockitoTestExecutionListener
 */
public class ResetMocksTestExecutionListener extends AbstractTestExecutionListener {

	private static final boolean MOCKITO_IS_PRESENT = ClassUtils.isPresent("org.mockito.MockSettings",
			ResetMocksTestExecutionListener.class.getClassLoader());

	@Override
	public int getOrder() {
		return Ordered.LOWEST_PRECEDENCE - 100;
	}

	@Override
	public void beforeTestMethod(TestContext testContext) throws Exception {
		if (MOCKITO_IS_PRESENT) {
			resetMocks(testContext.getApplicationContext(), MockReset.BEFORE);
		}
	}

	@Override
	public void afterTestMethod(TestContext testContext) throws Exception {
		if (MOCKITO_IS_PRESENT) {
			resetMocks(testContext.getApplicationContext(), MockReset.AFTER);
		}
	}

	private void resetMocks(ApplicationContext applicationContext, MockReset reset) {
		if (applicationContext instanceof ConfigurableApplicationContext) {
			resetMocks((ConfigurableApplicationContext) applicationContext, reset);
		}
	}

	private void resetMocks(ConfigurableApplicationContext applicationContext, MockReset reset) {
		//省略...
	}

}
```
## Spring测试场景
### 最小化Service测试
适用依赖比较少或者收敛于同一个包中的bean集合，只需关注逻辑运行结果的Bean测试

```java
package com.fit.fx.coll.trans.business.service;

import com.fit.fx.coll.trans.business.autoconfig.AlarmConfiguration;
import com.tencent.fit.autoconfig.FitAlarmAutoConfiguration;
import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.context.junit.jupiter.SpringJUnitConfig;

@SpringJUnitConfig(classes = {FitAlarmAutoConfiguration.class, AlarmConfiguration.class})
class JunitTester1 {

    @Autowired
    private AlarmService alarmService;

    @Test
    void testInjectAndLogic() {
        Assertions.assertNotNull(alarmService);
        alarmService.sendWarning(12345, "test error msg {0}", "FBI Warning!");
    }
}
```
### 中等链路测试
适用链路中等，从Service到ORM层或者只是RPC客户端调用测试
```java
package com.fit.fx.coll.trans.business;

import com.fit.fx.coll.trans.business.support.TestConfiguration;
import com.fit.fx.test.starter.PureLocalDataTest;

@PureLocalDataTest(classes = TestConfiguration.class, profiles = "test", mapperBasePackages = "com.fit.fx.coll.trans.business.mapper")
public abstract class LocalDataTestBase extends SpringContextTest {

}
```
```java
package com.fit.fx.coll.trans.business.service;

//imports

class CollStandardQueryServiceTest extends LocalDataTestBase {

    @Autowired
    private CollStandardQueryService collStandardQueryService;

    @Test
    @Sql("/sql/data_for_query.sql")
    void testStandardQueryDetail() throws JSONException {
        final QueryDetailReqDTO reqDTO = new QueryDetailReqDTO();
        reqDTO.setListid("COL0015900000541202202152160004137");
        reqDTO.setSpid("5900000541");
        final QueryDetailRspDTO queryDetailRspDTO = collStandardQueryService.queryCollOrderDetail(reqDTO);
        String expectedRsp = getJsonStr("/json/testStandardQueryDetail/order_detail.json");
        JSONAssert.assertEquals(expectedRsp, JsonUtils.toJson(queryDetailRspDTO), JSONCompareMode.LENIENT);
    }
}
```

### 全链路测试
适用长链路，从Service到ORM层以及RPC调用都涉及的Service测试，需要注入DAOBean和TRpcStub以及一些依赖的Service
```java
package com.fit.fx.coll.trans.business;

import com.fit.fx.coll.trans.business.support.TestConfiguration;
import com.fit.fx.test.starter.LocalDataAndTRpcTest;
import com.fit.fx.test.starter.configuration.EmbeddedRedisConfiguration;

@LocalDataAndTRpcTest(
        classes = {EmbeddedRedisConfiguration.class, TestConfiguration.class}, profiles = "test",
        mapperBasePackages = "com.fit.fx.coll.trans.business.mapper"
)
public class LocalDataAndTRpcTestBase extends SpringContextTest {

}
```
```java
package com.fit.fx.coll.trans.business.service;

//imports

class CollCallbackServiceTest extends LocalDataAndTRpcTestBase {

    @Autowired
    private CollCallbackService collCallbackService;

    @Autowired
    private CollOrderMapper collOrderMapper;

    @Autowired
    private ShopCollMapper shopCollMapper;

    @Autowired
    private OrderPayFailMapper orderPayFailMapper;

    @Autowired
    @Qualifier("noticeStringRedisTemplate")
    private StringRedisTemplate stringRedisTemplate;

    //unfreeze

    @Test
    @Sql("/sql/single_coll_order_for_unfreeze_callback.sql")
    @StubWithJson("/stub/fx_fitg_exchange_server.json")
    void testUnfreezeCallbackSuccess() {
        final String collListid = "mocked-coll-listid";
        final InnerSingleCollReqDTO reqDTO = new InnerSingleCollReqDTO();
        reqDTO.setCollListid(collListid);
        runWithMockGeneratedId(() -> collCallbackService.unfreeze(reqDTO));
        final CollOrderDO collOrderDO = collOrderMapper.selectByPrimaryKey(collListid);
        Assertions.assertEquals(CollState.COLL_STATE_UNFREEZE_SUCCEED, collOrderDO.getState());
        Assertions.assertEquals(4, collOrderDO.getVersion().intValue());
        final List<ShopCollDO> shopCollDOList = shopCollMapper.getShopCollListByCollListid(collListid);
        for (ShopCollDO shopCollDO : shopCollDOList) {
            if ("mocked-coll-listid0002".equals(shopCollDO.getId())) {
                Assertions.assertEquals(ShopCollState.SHOP_COLL_STATE_FROZEN_FAILED,
                        shopCollDO.getState());
                Assertions.assertNull(shopCollDO.getUnfreezeTime());
                Assertions.assertEquals(1, shopCollDO.getVersion().intValue());
            } else {
                Assertions.assertEquals(ShopCollState.SHOP_COLL_STATE_UNFREEZE_SUCCEED,
                        shopCollDO.getState());
                Assertions.assertNotNull(shopCollDO.getUnfreezeTime());
                Assertions.assertEquals(2, shopCollDO.getVersion().intValue());
            }
        }
    }
}
```

## 环境无关支持
### DB（MariaDB&Redis）
定义SpringBean，实现LifeCycle对应的生命周期方法，比如start、stop等，生命周期和Spring ApplicationContext保持一致，但是对于有多个配置的测试，Spring会维护多个Context并行，此时如果端口一样的话，势必会冲突，解决方案是Spring test的@DirtiesContext注解，让对应ApplicationContext结束其生命周期；
在测试过程中势必会有脏数据产生，因此自定义DirtyResource接口，在每次测试结束之后清理对应的DB（SpringExtension）

