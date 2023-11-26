## 服务端代码启动分析 加载配置

## 加载配置

加载配置看起来好像是一个很复杂的事情，启动时加载配置复用了hazelcast的加载配置逻辑。

```json
public class ServerExecuteCommand implements Command<ServerCommandArgs> {

    private final ServerCommandArgs serverCommandArgs;

    public ServerExecuteCommand(ServerCommandArgs serverCommandArgs) {
        this.serverCommandArgs = serverCommandArgs;
    }

    @Override
    public void execute() {
      //加载配置
        SeaTunnelConfig seaTunnelConfig = ConfigProvider.locateAndGetSeaTunnelConfig();
//设置集群名字
        if (StringUtils.isNotEmpty(serverCommandArgs.getClusterName())) {
            seaTunnelConfig.getHazelcastConfig().setClusterName(serverCommandArgs.getClusterName());
        }
//启动Hazelcast实例
        HazelcastInstanceFactory.newHazelcastInstance(
                seaTunnelConfig.getHazelcastConfig(),
                Thread.currentThread().getName(),
                new SeaTunnelNodeContext(seaTunnelConfig));
    }
}
```

## 解析配置

```java
public final class ConfigProvider {

    private ConfigProvider() {}
  
  //外部调用入口
    public static SeaTunnelConfig locateAndGetSeaTunnelConfig() {
        return locateAndGetSeaTunnelConfig(null);
    }

   public static SeaTunnelConfig locateAndGetSeaTunnelConfig(Properties properties) {
     //yaml配置解析入口
        YamlSeaTunnelConfigLocator yamlConfigLocator = new YamlSeaTunnelConfigLocator();
        SeaTunnelConfig config;
//英文注释写的很明白
        if (yamlConfigLocator.locateFromSystemProperty()) {
            // 1. Try loading YAML config if provided in system property
            config =
                    new YamlSeaTunnelConfigBuilder(yamlConfigLocator)
                            .setProperties(properties)
                            .build();
          //尝试直接获取文件或者从class.getClassLoader().getResource(configFileName)获取
        } else if (yamlConfigLocator.locateInWorkDirOrOnClasspath()) {
            // 2. Try loading YAML config from the working directory or from the classpath
            config =
                    new YamlSeaTunnelConfigBuilder(yamlConfigLocator)
                            .setProperties(properties)
                            .build();
        } else {
            // 3. Loading the default YAML configuration file
          //否则使用默认的配置文件
            yamlConfigLocator.locateDefault();
            config =
                    new YamlSeaTunnelConfigBuilder(yamlConfigLocator)
                            .setProperties(properties)
                            .build();
        }
        return config;
    }

    @NonNull public static ClientConfig locateAndGetClientConfig() {
        validateSuffixInSystemProperty(SYSPROP_CLIENT_CONFIG);

        ClientConfig config;
        YamlClientConfigLocator yamlConfigLocator = new YamlClientConfigLocator();

        if (yamlConfigLocator.locateFromSystemProperty()) {
            // 1. Try loading config if provided in system property, and it is an YAML file
            config = new YamlClientConfigBuilder(yamlConfigLocator.getIn()).build();
        } else if (yamlConfigLocator.locateInWorkDirOrOnClasspath()) {
            // 2. Try loading YAML config from the working directory or from the classpath
            config = new YamlClientConfigBuilder(yamlConfigLocator.getIn()).build();
        } else {
            // 3. Loading the default YAML configuration file
            yamlConfigLocator.locateDefault();
            config = new YamlClientConfigBuilder(yamlConfigLocator.getIn()).build();
        }
        return config;
    }
}
```

首先尝试从系统属性加载，然后尝试从工作目录或者classpath加载，最后尝试直接从classpath加载给定的默认配置文件。

## 提交任务

submit任务和配置到主节点

然后主节点对任务进行拆分，将任务的执行动作抽象出operation发送到member
