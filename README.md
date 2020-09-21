#### 导入maven依赖包

```xml
<dependency>
   <groupId>com.netflix.ribbon</groupId>
   <artifactId>ribbon</artifactId>
   <version>2.2.2</version>
</dependency>
<dependency>
   <groupId>com.netflix.eureka</groupId>
   <artifactId>eureka-client</artifactId>
   <version>1.7.0</version>
</dependency>
<dependency>
   <groupId>com.netflix.feign</groupId>
   <artifactId>feign-core</artifactId>
   <version>8.18.0</version>
</dependency>
<dependency>
   <groupId>com.netflix.feign</groupId>
   <artifactId>feign-jackson</artifactId>
   <version>8.17.0</version>
</dependency>
<dependency>
   <groupId>com.netflix.ribbon</groupId>
   <artifactId>ribbon-core</artifactId>
   <version>2.1.2</version>
</dependency>
<dependency>
   <groupId>com.netflix.ribbon</groupId>
   <artifactId>ribbon-loadbalancer</artifactId>
   <version>2.0.0</version>
</dependency>
<dependency>
   <groupId>com.netflix.ribbon</groupId>
   <artifactId>ribbon-eureka</artifactId>
   <version>2.1.2</version>
</dependency>
<dependency>
   <groupId>com.netflix.archaius</groupId>
   <artifactId>archaius-core</artifactId>
   <version>0.7.6</version>
</dependency>

<dependency>
   <groupId>com.netflix.curator</groupId>
   <artifactId>curator-recipes</artifactId>
   <version>1.3.3</version>
   <exclusions>
      <exclusion>
         <groupId>org.slf4j</groupId>
         <artifactId>slf4j-log4j12</artifactId>
      </exclusion>
   </exclusions>
</dependency>
```

#### 配置文件

```properties
#本身服务是否注册在eureka上面
eureka.registration.enabled=false

#eureka参数
eureka.preferSameZone=true
eureka.shouldUseDns=false
eureka.serviceUrl.default=http://10.153.99.56:8761/eureka/
eureka.decoderName=JacksonJson

xxx.ribbon.NIWSServerListClassName=com.netflix.niws.loadbalancer.DiscoveryEnabledNIWSServerList
xxx.ribbon.ServerListRefreshInterval=60000
xxx.ribbon.DeploymentContextBasedVipAddresses=${MS_SERVICE_NAME}
```

#### 地址和端口的实体类

```java
public class AlanServiceAddress {
    private int port;
    private String host;

    public AlanServiceAddress() {}

    public AlanServiceAddress(int port, String host) {
        this.port = port;
        this.host = host;
    }


    public int getPort() {
        return port;
    }

    public void setPort(int port) {
        this.port = port;
    }

    public String getHost() {
        return host;
    }

    public void setHost(String host) {
        this.host = host;
    }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder(15 + host.length());
        sb.append("http://").append(host).append(":").append(port).append("/");
        return sb.toString();
    }
}
```

## 初始化Ribbon、注册Eureka

```java
import com.netflix.appinfo.MyDataCenterInstanceConfig;
import com.netflix.client.ClientFactory;
import com.netflix.config.ConfigurationManager;
import com.netflix.discovery.DefaultEurekaClientConfig;
import com.netflix.discovery.DiscoveryManager;
import com.netflix.loadbalancer.DynamicServerListLoadBalancer;
import com.netflix.loadbalancer.RoundRobinRule;
import com.netflix.loadbalancer.Server;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.InputStream;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Properties;

@SuppressWarnings("deprecation")
public class AlanServiceAddressSelector {
    public static final String RIBBON_CONFIG_FILE_NAME = "normandy.properties";

    private static final Logger log = LoggerFactory.getLogger(AlanServiceAddressSelector.class);

    private static RoundRobinRule chooseRule = new RoundRobinRule();

    static {
        try {
            Properties properties = new Properties();
            InputStream inputStream = Thread.currentThread().getContextClassLoader()
                    .getResourceAsStream(RIBBON_CONFIG_FILE_NAME);
            properties.load(inputStream);
            ConfigurationManager.loadProperties(properties);
        } catch (Exception e) {
            e.printStackTrace();
            throw new IllegalStateException("ribbon初始化失败");
        }
        DiscoveryManager.getInstance().initComponent(new MyDataCenterInstanceConfig(),
                new DefaultEurekaClientConfig());
    }

    @SuppressWarnings("rawtypes")
    public static AlanServiceAddress selectOne(String clientName) {
        DynamicServerListLoadBalancer lb = (DynamicServerListLoadBalancer) ClientFactory
                .getNamedLoadBalancer(clientName);
        Server selected = chooseRule.choose(lb, null);
        if (null == selected) {
            log.warn("服务{}没有可用地址", clientName);
            return null;
        }
        log.debug("服务{}选择结果:{}", clientName, selected);
        return new AlanServiceAddress(selected.getPort(), selected.getHost());
    }

    @SuppressWarnings("rawtypes")
    public static List<AlanServiceAddress> selectAvailableServers(String clientName) {
        DynamicServerListLoadBalancer lb = (DynamicServerListLoadBalancer) ClientFactory
                .getNamedLoadBalancer(clientName);
        List<Server> serverList = lb.getServerList(true);
        if (serverList.isEmpty()) {
            log.warn("服务{}没有可用地址", clientName);
            return Collections.emptyList();
        }
        log.debug("服务{}所有选择结果:{}", clientName, serverList);
        List<AlanServiceAddress> addrList = new ArrayList<>(serverList.size());
        for (Server server : serverList) {
            addrList.add(new AlanServiceAddress(server.getPort(), server.getHost()));
        }
        return addrList;
    }
}
```

#### 创建feign的bean

```java
import com.cmcc.normandy.client.feign.service.ISpringCloudAppService;
import com.cmcc.normandy.client.feign.service.ISpringCloudDevelopService;
import com.cmcc.normandy.client.feign.service.ISpringCloudSourceService;
import com.cmcc.normandy.common.utils.NormandyConfig;
import feign.Feign;
import feign.Feign.Builder;
import feign.jackson.JacksonDecoder;
import feign.jackson.JacksonEncoder;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.List;

@Configuration
public class NonCloudFeignConfig {
    private static final Logger log = LoggerFactory.getLogger(NonCloudFeignConfig.class);
    private static final Builder BUILDER = Feign.builder().encoder(new JacksonEncoder())
            .decoder(new JacksonDecoder());
    public static String feginUrl;
    
    static {
        //NormandyConfig 解析properties配置类
       String serviceName = NormandyConfig.get("MS_SERVICE_NAME", "dw-sourceid");
        List<AlanServiceAddress> addressList = AlanServiceAddressSelector
                .selectAvailableServers(serviceName);
        feginUrl = AlanServiceAddressSelector.selectAvailableServers(serviceName)
                .get(new java.util.Random().nextInt(addressList.size())).toString();
        log.info("feginUrl = {}", feginUrl);
    }


    @Bean
    public ISpringCloudDevelopService createCloudDevelopService() {
        return BUILDER.target(ISpringCloudDevelopService.class, feginUrl);
    }


}
```

#### bean的实例

```java
import com.alibaba.fastjson.JSONObject;
import feign.Headers;
import feign.RequestLine;
@Headers({ "Content-Type: application/json", "Accept: application/json" })
public interface ISpringCloudDevelopService {

    @RequestLine("POST /develop/register")
    JSONObject registerDeveloper(JSONObject jsonObject);

}
```

