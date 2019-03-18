---
title: spring-cloud-gateway基于redis的动态路由
date: 2019-03-17 23:48:20
categories:
  - tech
tags:
  - spring-cloud-gateway
---

### 一、增加路由管理模块

#### 1.1 启动时加载数据库中配置文件到`Redis`

```java
/**
 * @author dudiao
 *
 * <p>
 * 容器启动后保存配置文件里面的路由信息到Redis
 */
@Slf4j
@Configuration
@AllArgsConstructor
public class DynamicRouteInitRunner {
    private final RedisTemplate redisTemplate;
    private final SysRouteConfService routeConfService;

    @Async
    @Order
    @EventListener({WebServerInitializedEvent.class, DynamicRouteInitEvent.class})
    public void initRoute() {
        Boolean result = redisTemplate.delete(CommonConstants.ROUTE_KEY);
        log.info("初始化网关路由 {} ", result);

        routeConfService.routes().forEach(route -> {
            RouteDefinitionVo vo = new RouteDefinitionVo();
            vo.setRouteName(route.getRouteName());
            vo.setId(route.getRouteId());
            vo.setUri(URI.create(route.getUri()));
            vo.setOrder(route.getOrder());

            JSONArray filterObj = JSONUtil.parseArray(route.getFilters());
            vo.setFilters(filterObj.toList(FilterDefinition.class));
            JSONArray predicateObj = JSONUtil.parseArray(route.getPredicates());
            vo.setPredicates(predicateObj.toList(PredicateDefinition.class));

            log.info("加载路由ID：{},{}", route.getRouteId(), vo);
            redisTemplate.setHashValueSerializer(new Jackson2JsonRedisSerializer<>(RouteDefinitionVo.class));
            redisTemplate.opsForHash().put(CommonConstants.ROUTE_KEY, route.getRouteId(), vo);
        });
        log.info("初始化网关路由结束 ");
    }

    /**
     * 配置文件设置为空redis 加载的为准
     *
     * @return
     */
    @Bean
    public PropertiesRouteDefinitionLocator propertiesRouteDefinitionLocator() {
        return new PropertiesRouteDefinitionLocator(new GatewayProperties());
    }
}
```

#### 1.2 路由管理模块：获取路由和更新路由

参考GatewayControllerEndpoint实现

```java
@Slf4j
@AllArgsConstructor
@Service("sysRouteConfService")
public class SysRouteConfServiceImpl extends ServiceImpl<SysRouteConfMapper, SysRouteConf> implements SysRouteConfService {
	private final RedisTemplate redisTemplate;
	private final ApplicationEventPublisher applicationEventPublisher;


	/**
	 * 获取全部路由
	 * <p>
	 * RedisRouteDefinitionWriter.java
	 * PropertiesRouteDefinitionLocator.java
	 *
	 * @return
	 */
	@Override
	public List<SysRouteConf> routes() {
		return baseMapper.selectList(Wrappers.emptyWrapper());
	}

	/**
	 * 更新路由信息
	 *
	 * @param routes 路由信息
	 * @return
	 */
	@Override
	@Transactional(rollbackFor = Exception.class)
	public Mono<Void> updateRoutes(JSONArray routes) {
		// 清空Redis 缓存
		Boolean result = redisTemplate.delete(CommonConstants.ROUTE_KEY);
		log.info("清空网关路由 {} ", result);

		// 遍历修改的routes，保存到Redis
		List<RouteDefinitionVo> routeDefinitionVoList = new ArrayList<>();
		routes.forEach(value -> {
			log.info("更新路由 ->{}", value);
			RouteDefinitionVo vo = new RouteDefinitionVo();
			Map<String, Object> map = (Map) value;

			Object id = map.get("routeId");
			if (id != null) {
				vo.setId(String.valueOf(id));
			}

			Object routeName = map.get("routeName");
			if (routeName != null) {
				vo.setRouteName(String.valueOf(routeName));
			}

			Object predicates = map.get("predicates");
			if (predicates != null) {
				JSONArray predicatesArray = (JSONArray) predicates;
				List<PredicateDefinition> predicateDefinitionList =
					predicatesArray.toList(PredicateDefinition.class);
				vo.setPredicates(predicateDefinitionList);
			}

			Object filters = map.get("filters");
			if (filters != null) {
				JSONArray filtersArray = (JSONArray) filters;
				List<FilterDefinition> filterDefinitionList
					= filtersArray.toList(FilterDefinition.class);
				vo.setFilters(filterDefinitionList);
			}

			Object uri = map.get("uri");
			if (uri != null) {
				vo.setUri(URI.create(String.valueOf(uri)));
			}

			Object order = map.get("order");
			if (order != null) {
				vo.setOrder(Integer.parseInt(String.valueOf(order)));
			}

			redisTemplate.setHashValueSerializer(new Jackson2JsonRedisSerializer<>(RouteDefinitionVo.class));
			redisTemplate.opsForHash().put(CommonConstants.ROUTE_KEY, vo.getId(), vo);
			routeDefinitionVoList.add(vo);
		});

		// 逻辑删除全部
		SysRouteConf condition = new SysRouteConf();
		condition.setDelFlag(CommonConstants.STATUS_NORMAL);
		this.remove(new UpdateWrapper<>(condition));

		//插入生效路由
		List<SysRouteConf> routeConfList = routeDefinitionVoList.stream().map(vo -> {
			SysRouteConf routeConf = new SysRouteConf();
			routeConf.setRouteId(vo.getId());
			routeConf.setRouteName(vo.getRouteName());
			routeConf.setFilters(JSONUtil.toJsonStr(vo.getFilters()));
			routeConf.setPredicates(JSONUtil.toJsonStr(vo.getPredicates()));
			routeConf.setOrder(vo.getOrder());
			routeConf.setUri(vo.getUri().toString());
			return routeConf;
		}).collect(Collectors.toList());
		this.saveBatch(routeConfList);
		log.debug("更新网关路由结束 ");

		this.applicationEventPublisher.publishEvent(new RefreshRoutesEvent(this));
		return Mono.empty();
	}

}
```

### 二、定义 Redis 存储器

继承`RouteDefinitionRepository`，重写`save / delete / getRouteDefinitions`方法。

```java
@Slf4j
@Component
@AllArgsConstructor
public class RedisRouteDefinitionWriter implements RouteDefinitionRepository {
	private final RedisTemplate redisTemplate;

	@Override
	public Mono<Void> save(Mono<RouteDefinition> route) {
		return route.flatMap(r -> {
			RouteDefinitionVo vo = new RouteDefinitionVo();
			BeanUtils.copyProperties(r, vo);
			log.info("保存路由信息{}", vo);
			redisTemplate.setKeySerializer(new StringRedisSerializer());
			redisTemplate.opsForHash().put(CommonConstants.ROUTE_KEY, r.getId(), vo);
			return Mono.empty();
		});
	}

	@Override
	public Mono<Void> delete(Mono<String> routeId) {
		routeId.subscribe(id -> {
			log.info("删除路由信息{}", id);
			redisTemplate.setKeySerializer(new StringRedisSerializer());
			redisTemplate.opsForHash().delete(CommonConstants.ROUTE_KEY, id);
		});
		return Mono.empty();
	}


	/**
	 * 动态路由入口
	 *
	 * @return
	 */
	@Override
	public Flux<RouteDefinition> getRouteDefinitions() {
		redisTemplate.setKeySerializer(new StringRedisSerializer());
		redisTemplate.setHashValueSerializer(new Jackson2JsonRedisSerializer<>(RouteDefinitionVo.class));
		List<RouteDefinitionVo> values = redisTemplate.opsForHash().values(CommonConstants.ROUTE_KEY);
		List<RouteDefinition> definitionList = new ArrayList<>();
		values.forEach(vo -> {
			RouteDefinition routeDefinition = new RouteDefinition();
			BeanUtils.copyProperties(vo, routeDefinition);
			definitionList.add(vo);
		});
		log.debug("redis 中路由定义条数： {}， {}", definitionList.size(), definitionList);
		return Flux.fromIterable(definitionList);
	}
}
```
