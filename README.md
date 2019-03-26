<a href=#index1>1. 安装redis</a>  
<a href=#index2>2. 创建节点</a>  
<a href=#index3>3. 搭建集群</a>  
<a href=#index4>4. 测试集群</a>  
<a href=#index5>5. Redis Cluster、Lettuce与Spring集成</a>  
<a href=#index6>6. 参考资料</a>


## <span id=index1>安装redis</span>
```
#源码编译需要安装gcc
sudo apt install gcc
wget http://download.redis.io/releases/redis-5.0.4.tar.gz
tar xzf redis-5.0.4.tar.gz
cd redis-5.0.4
make
```


将redis-server添加到环境变量中:
```
export PATH=${PATH}:/path-to-redis/src
```


## <span id=index2>创建节点</span>
redis集群可部署在同一台机器上,以多实例来弥补redis单线程的不足,也可部署在不同的机器上做水平扩展.  
创建6个文件夹作为redis集群的节点(三主三从),并以端口号命名(7000-70005)  
创建redis配置文件`redis.conf`,启用redis集群配置:  
```
port 7000
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```


将`redis.conf`复制到各个目录下:
``` bash
find /path-to-redis-cluster -type d | xargs -n 1 cp /home/ganguo/redis.conf
```
修改各个`redis.conf`中的端口号,与文件夹名保持一致.  
分别在各个目录中启动redis实例:
``` bash
cd path-to-redis-cluster
find $(pwd) ! -path $(pwd) -type d|xargs -I {} -n 1 sh -c 'cd {}&&redis-server {}/redis.conf&'
``` 
启动成功后会在各个redis实例的目录下生成`nodes.conf`文件,这个文件用于redis维护集群中的节点信息,不需要人为修改.  


## <span id=index3>搭建集群</span>
``` bash
#redis3 或 redis4使用redis-trib.rb(位于redis源码的src文件夹中)进行配置
redis-trib.rb --replicas create 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005
#在redis5中,redis-trib.rb已经整合进了redis-cli中
redis-cli --cluster create 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 --cluster-replicas 1
```
若应用与redis集群不在同一台机器上,应将`127.0.0.1`修改为各个redis实例的公网ip或域名.  
redis集群不支持使用密码,不可在`redis.conf`选项中指定`requirepass`选项,而redis本身不支持ip白名单功能,可考虑使用防火墙配置白名单:
``` bash
for i in {7000..7005}
do
    sudo ufw allow from 192.168.1.1 to any port ${i} #将192.168.1.1替换为应用部署的ip
done
```


## <span id=index4>测试集群</span>
往集群中新增key:
```
redis-cli -c -p 7000
set hello world
set foo bar
#输出
127.0.0.1:7000> set hello world
OK
127.0.0.1:7000> set foo bar
-> Redirected to slot [12182] located at 127.0.0.1:7002
OK
```
根据输出内容可以知道`hello`的值被存储在7000端口的实例上,而`foo`的值被存储在7002端口的实例上,说明redis cluster工作正常.  
`redis-cli`的`-c`选项表示启用集群模式.


## <span id=index5>Redis Cluster、Lettuce与Spring集成</span>
在properties文件中加入redis集群节点配置:
``` properties
#只需要添加集群中的任一个节点信息即可,redis可通过任一节点获取到集群中的其他节点信息
spring.redis.cluster[0]=127.0.0.1:7000
spring.redis.cluster[1]=127.0.0.1:7001
spring.redis.cluster[2]=127.0.0.1:7002
```
Spring Bean配置:
``` java
@Setter
@Getter
@ConfigurationProperties(prefix = "spring.redis")
@Configuration
public class RedisConfig {
    private String host;
    private int port;

    @Bean(destroyMethod = "shutdown")
    ClientResources clientResources() {
        return DefaultClientResources.create();
    }

    @Bean
    public RedisConnectionFactory redisClusterConnectionFactory() {
        return new LettuceConnectionFactory(new RedisClusterConfiguration(getClusterNodes()));
    }

    @Bean(destroyMethod = "shutdown")
    RedisClient provideRedisClient(ClientResources clientResources) {
        RedisURI redisURI = RedisURI.create(host, port);
        return RedisClient.create(clientResources, redisURI);
    }

    @Bean
    public StringRedisSerializer provideStringRedisSerializer() {
        return new StringRedisSerializer(StandardCharsets.UTF_8);
    }

    @Bean("springSessionDefaultRedisSerializer")
    public GenericJackson2JsonRedisSerializer valueSerializer(ObjectMapper objectMapper) {
        objectMapper = objectMapper.copy()
                .enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        return new GenericJackson2JsonRedisSerializer(objectMapper);
    }

    @Bean
    public RedisTemplate<String, Object> clusterRedisTemplate(RedisConnectionFactory connectionFactory,
                                                              StringRedisSerializer stringRedisSerializer,
                                                              GenericJackson2JsonRedisSerializer redisSerializer) {
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setKeySerializer(stringRedisSerializer);
        redisTemplate.setHashKeySerializer(redisSerializer);
        redisTemplate.setValueSerializer(redisSerializer);
        redisTemplate.setHashValueSerializer(redisSerializer);
        redisTemplate.setConnectionFactory(connectionFactory);
        redisTemplate.setEnableTransactionSupport(false);
        redisTemplate.afterPropertiesSet();
        return redisTemplate;
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.redis.cluster")
    public List<String> getClusterNodes() {
        return new ArrayList<>();
    }
}
```


## <span id=index6>参考资料</span>
- [Redis集群教程](http://www.redis.cn/topics/cluster-tutorial.html)
- [Redis cluster tutorial](https://redis.io/topics/cluster-tutorial)
- [Spring Support](https://github.com/lettuce-io/lettuce-core/wiki/Spring-Support)

