# @Schedule使用技巧+源码分析

### 先说结论

> 使用@EnableScheduling + @Scheduled 注解可以快速的实现定时任务的需求，但是Spring提供的方式默认使用的是单线程，若在任务耗时长，任务多的项目中有任务彼此干扰阻塞执行的风险。所以平时使用还需要做好多线程的配置。配置自己的线程池为线程取一个独特的业务名字也便于排查问题。

### 复现问题

**JDK8 + Spring5.3.24/Springboot2.3.7**

在启动类上加上@EnableScheduling注解，这是支持定时任务的注解。

```Java
@SpringBootApplication
@EnableScheduling
public class Demo1Application {

    public static void main(String[] args) {
        SpringApplication.run(Demo1Application.class, args);
    }

}
```

测试service

```Java
@Service
public class TaskService {
    AtomicInteger atomicInteger = new AtomicInteger(1);

    @Scheduled(cron = "0/1 * * * * ? ")
    public void aRun() {
        while (true) {
            try {
                System.out.println("aRun:" + LocalDateTime.now() + "---------");
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }
    }

    @Scheduled(cron = "0/1 * * * * ? ")
    public void bRun() {
        while (true) {
            try {
                System.out.println("bRun:" + LocalDateTime.now() + "---------");
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }
    }
}
```

在测试类中我声明了两个定时任务，每秒执行一次，然后每个任务都是循环体不会退出。那么最终的结果就是只会有一个任务执行，另外一个任务永远也不会执行。

![img](http://www.xiewenshi.fans/blogimg/scheduledemo.PNG)

### 解决办法

```Java
@Configuration
public class ScheduleTaskConfig implements SchedulingConfigurer {
    @Override
    //注册自己的任务执行线程池
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        int processors = Runtime.getRuntime().availableProcessors();
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(processors);
        scheduler.initialize();
        scheduler.setThreadFactory(new NamedThreadFactory("scheduleTask-", false));
        scheduler.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardOldestPolicy());
        taskRegistrar.setTaskScheduler(scheduler);
    }
}
```

### 源码分析

**以spring5.3.24为例。**

需要使用@Scheduled的时候 必须要在配置类或者启动类加上 @EnableScheduling 注解才可以。

那么先从@EnableScheduling开始。

#### EnableScheduling

```Java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import(SchedulingConfiguration.class)
@Documented
public @interface EnableScheduling {

}
```

在EnableScheduling中Import了SchedulingConfiguration，Import注解中的class会被快速的注入到Spring的容器，见名知意，就知道这是定时调度相关的配置类，也是为什么要使用了EnableScheduling注解才能开启定时任务。

#### SchedulingConfiguration

```Java
@Configuration
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class SchedulingConfiguration {

   @Bean(name = TaskManagementConfigUtils.SCHEDULED_ANNOTATION_PROCESSOR_BEAN_NAME)
   @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
   public ScheduledAnnotationBeanPostProcessor scheduledAnnotationProcessor() {
      return new ScheduledAnnotationBeanPostProcessor();
   }

}
```

这个配置类也就是做了注册处理Schedule注解的后置处理器ScheduledAnnotationBeanPostProcessor。

#### ScheduledAnnotationBeanPostProcessor

> 当一个类实现了BeanPostProcessor之后，spring 容器在初始化系统的每个bean的时候都会调用这个类实现的BeanPostProcessor的接口方法，并把bean对像和名称作为参数传给这个类对像。

当我们的Service类被spring初始化完成之后，会调用BeanPostProcessor接口的postProcessAfterInitialization方法。

##### postProcessAfterInitialization

>    **1:****遍历了每个bean的方法，并找出使用了@Scheduled的方法，**   **2:****调用processScheduled进行处理被注解了的方法**

```Java
class ScheduledAnnotationBeanPostProcessor{
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) {
   if (bean instanceof AopInfrastructureBean || bean instanceof TaskScheduler ||
         bean instanceof ScheduledExecutorService) {
      // Ignore AOP infrastructure such as scoped proxies.
      return bean;
   }

   Class<?> targetClass = AopProxyUtils.ultimateTargetClass(bean);
   //遍历了每个bean的方法，并找出使用了@Scheduled的方法，
   //调用processScheduled进行处理被注解了的方法
   if (!this.nonAnnotatedClasses.contains(targetClass) &&
         AnnotationUtils.isCandidateClass(targetClass, Arrays.asList(Scheduled.class, Schedules.class))) {
      Map<Method, Set<Scheduled>> annotatedMethods = MethodIntrospector.selectMethods(targetClass,
            (MethodIntrospector.MetadataLookup<Set<Scheduled>>) method -> {
               Set<Scheduled> scheduledMethods = AnnotatedElementUtils.getMergedRepeatableAnnotations(
                     method, Scheduled.class, Schedules.class);
               return (!scheduledMethods.isEmpty() ? scheduledMethods : null);
            });
      if (annotatedMethods.isEmpty()) {
         this.nonAnnotatedClasses.add(targetClass);
         if (logger.isTraceEnabled()) {
            logger.trace("No @Scheduled annotations found on bean class: " + targetClass);
         }
      }
      else {
      //调用processScheduled进行处理被注解了的方法
         // Non-empty set of methods
         annotatedMethods.forEach((method, scheduledMethods) ->
               scheduledMethods.forEach(scheduled -> processScheduled(scheduled, method, bean)));
         if (logger.isTraceEnabled()) {
            logger.trace(annotatedMethods.size() + " @Scheduled methods processed on bean '" + beanName +
                  "': " + annotatedMethods);
         }
      }
   }
   return bean;
}
}
```

##### processScheduled

>  **根据注解的参数不同 生成不同类型的任务 收集起来保存起来 但是还没将任务丢给线程池执行**
>
>  **同时也交给了ScheduledTaskRegistrar 也就是为什么后面ScheduledTaskRegistrar会有所有的任务**

```Java
protected void processScheduled(Scheduled scheduled, Method method, Object bean) {
   try {
      Runnable runnable = createRunnable(bean, method);
      boolean processedSchedule = false;
      String errorMessage =
            "Exactly one of the 'cron', 'fixedDelay(String)', or 'fixedRate(String)' attributes is required";
            //任务集合
      Set<ScheduledTask> tasks = new LinkedHashSet<>(4);

      // Determine initial delay
      long initialDelay = scheduled.initialDelay();
      String initialDelayString = scheduled.initialDelayString();
      if (StringUtils.hasText(initialDelayString)) {
         Assert.isTrue(initialDelay < 0, "Specify 'initialDelay' or 'initialDelayString', not both");
         if (this.embeddedValueResolver != null) {
            initialDelayString = this.embeddedValueResolver.resolveStringValue(initialDelayString);
         }
         if (StringUtils.hasLength(initialDelayString)) {
            try {
               initialDelay = parseDelayAsLong(initialDelayString);
            }
            catch (RuntimeException ex) {
               throw new IllegalArgumentException(
                     "Invalid initialDelayString value \"" + initialDelayString + "\" - cannot parse into long");
            }
         }
      }
      /* 以下是
      根据注解的参数不同 生成不同类型的任务
      放到tasks集合中
      */

      // Check cron expression
      String cron = scheduled.cron();
      if (StringUtils.hasText(cron)) {
         String zone = scheduled.zone();
         if (this.embeddedValueResolver != null) {
            cron = this.embeddedValueResolver.resolveStringValue(cron);
            zone = this.embeddedValueResolver.resolveStringValue(zone);
         }
         if (StringUtils.hasLength(cron)) {
            Assert.isTrue(initialDelay == -1, "'initialDelay' not supported for cron triggers");
            processedSchedule = true;
            if (!Scheduled.CRON_DISABLED.equals(cron)) {
               TimeZone timeZone;
               if (StringUtils.hasText(zone)) {
                  timeZone = StringUtils.parseTimeZoneString(zone);
               }
               else {
                  timeZone = TimeZone.getDefault();
               }
               // 同时也交给了ScheduledTaskRegistrar 
               //也就是为什么后面ScheduledTaskRegistrar会有所有的任务
               //后面分类处理差不多的逻辑
               tasks.add(this.registrar.scheduleCronTask(new CronTask(runnable, new CronTrigger(cron, timeZone))));
            }
         }
      }

      // At this point we don't need to differentiate between initial delay set or not anymore
      if (initialDelay < 0) {
         initialDelay = 0;
      }

      // Check fixed delay
      long fixedDelay = scheduled.fixedDelay();
      if (fixedDelay >= 0) {
         Assert.isTrue(!processedSchedule, errorMessage);
         processedSchedule = true;
         tasks.add(this.registrar.scheduleFixedDelayTask(new FixedDelayTask(runnable, fixedDelay, initialDelay)));
      }
      String fixedDelayString = scheduled.fixedDelayString();
      if (StringUtils.hasText(fixedDelayString)) {
         if (this.embeddedValueResolver != null) {
            fixedDelayString = this.embeddedValueResolver.resolveStringValue(fixedDelayString);
         }
         if (StringUtils.hasLength(fixedDelayString)) {
            Assert.isTrue(!processedSchedule, errorMessage);
            processedSchedule = true;
            try {
               fixedDelay = parseDelayAsLong(fixedDelayString);
            }
            catch (RuntimeException ex) {
               throw new IllegalArgumentException(
                     "Invalid fixedDelayString value \"" + fixedDelayString + "\" - cannot parse into long");
            }
            tasks.add(this.registrar.scheduleFixedDelayTask(new FixedDelayTask(runnable, fixedDelay, initialDelay)));
         }
      }

      // Check fixed rate
      long fixedRate = scheduled.fixedRate();
      if (fixedRate >= 0) {
         Assert.isTrue(!processedSchedule, errorMessage);
         processedSchedule = true;
         tasks.add(this.registrar.scheduleFixedRateTask(new FixedRateTask(runnable, fixedRate, initialDelay)));
      }
      String fixedRateString = scheduled.fixedRateString();
      if (StringUtils.hasText(fixedRateString)) {
         if (this.embeddedValueResolver != null) {
            fixedRateString = this.embeddedValueResolver.resolveStringValue(fixedRateString);
         }
         if (StringUtils.hasLength(fixedRateString)) {
            Assert.isTrue(!processedSchedule, errorMessage);
            processedSchedule = true;
            try {
               fixedRate = parseDelayAsLong(fixedRateString);
            }
            catch (RuntimeException ex) {
               throw new IllegalArgumentException(
                     "Invalid fixedRateString value \"" + fixedRateString + "\" - cannot parse into long");
            }
            tasks.add(this.registrar.scheduleFixedRateTask(new FixedRateTask(runnable, fixedRate, initialDelay)));
         }
      }

      // Check whether we had any attribute set
      Assert.isTrue(processedSchedule, errorMessage);

      // Finally register the scheduled tasks
      //最后收集到全局变量scheduledTasks中
      synchronized (this.scheduledTasks) {
         Set<ScheduledTask> regTasks = this.scheduledTasks.computeIfAbsent(bean, key -> new LinkedHashSet<>(4));
         regTasks.addAll(tasks);
      }
   }
   catch (IllegalArgumentException ex) {
      throw new IllegalStateException(
            "Encountered invalid @Scheduled method '" + method.getName() + "': " + ex.getMessage());
   }
}
```

##### ApplicationListener<ContextRefreshedEvent>

> ScheduledAnnotationBeanPostProcessor 监听 ContextRefreshedEvent 事件会在Spring容器初始化完成会触发void onApplicationEvent(E event)方法。来做收尾工作

```Java
@Override
public void onApplicationEvent(ContextRefreshedEvent event) {
   if (event.getApplicationContext() == this.applicationContext) {
      // Running in an ApplicationContext -> register tasks this late...
      // giving other ContextRefreshedEvent listeners a chance to perform
      // their work at the same time (e.g. Spring Batch's job registration).
      finishRegistration();
   }
}
```

> 找到SchedulingConfigurer的所有实现类，遍历实现类，注册到SchedulingConfigurer中
>
> 在配置中 我们注册了自己的线程池 那么如果没有注册呢？需要看一下registrar的逻辑

```Java
private void finishRegistration() {
   if (this.scheduler != null) {
      this.registrar.setScheduler(this.scheduler);
   }

   if (this.beanFactory instanceof ListableBeanFactory) {
   //找到SchedulingConfigurer的所有实现类，遍历实现类，注册到SchedulingConfigurer中
   //这也是我们动态配置的拓展点
   //在配置中 我们注册了自己的线程池 那么如果没有注册呢？需要看一下registrar的逻辑
      Map<String, SchedulingConfigurer> beans =
            ((ListableBeanFactory) this.beanFactory).getBeansOfType(SchedulingConfigurer.class);
      List<SchedulingConfigurer> configurers = new ArrayList<>(beans.values());
      AnnotationAwareOrderComparator.sort(configurers);
      for (SchedulingConfigurer configurer : configurers) {
         configurer.configureTasks(this.registrar);
      }
   }

   if (this.registrar.hasTasks() && this.registrar.getScheduler() == null) {
      Assert.state(this.beanFactory != null, "BeanFactory must be set to find scheduler by type");
      try {
         // Search for TaskScheduler bean...
         this.registrar.setTaskScheduler(resolveSchedulerBean(this.beanFactory, TaskScheduler.class, false));
      }
      catch (NoUniqueBeanDefinitionException ex) {
         if (logger.isTraceEnabled()) {
            logger.trace("Could not find unique TaskScheduler bean - attempting to resolve by name: " +
                  ex.getMessage());
         }
         try {
            this.registrar.setTaskScheduler(resolveSchedulerBean(this.beanFactory, TaskScheduler.class, true));
         }
         catch (NoSuchBeanDefinitionException ex2) {
            if (logger.isInfoEnabled()) {
               logger.info("More than one TaskScheduler bean exists within the context, and " +
                     "none is named 'taskScheduler'. Mark one of them as primary or name it 'taskScheduler' " +
                     "(possibly as an alias); or implement the SchedulingConfigurer interface and call " +
                     "ScheduledTaskRegistrar#setScheduler explicitly within the configureTasks() callback: " +
                     ex.getBeanNamesFound());
            }
         }
      }
      catch (NoSuchBeanDefinitionException ex) {
         if (logger.isTraceEnabled()) {
            logger.trace("Could not find default TaskScheduler bean - attempting to find ScheduledExecutorService: " +
                  ex.getMessage());
         }
         // Search for ScheduledExecutorService bean next...
         try {
            this.registrar.setScheduler(resolveSchedulerBean(this.beanFactory, ScheduledExecutorService.class, false));
         }
         catch (NoUniqueBeanDefinitionException ex2) {
            if (logger.isTraceEnabled()) {
               logger.trace("Could not find unique ScheduledExecutorService bean - attempting to resolve by name: " +
                     ex2.getMessage());
            }
            try {
               this.registrar.setScheduler(resolveSchedulerBean(this.beanFactory, ScheduledExecutorService.class, true));
            }
            catch (NoSuchBeanDefinitionException ex3) {
               if (logger.isInfoEnabled()) {
                  logger.info("More than one ScheduledExecutorService bean exists within the context, and " +
                        "none is named 'taskScheduler'. Mark one of them as primary or name it 'taskScheduler' " +
                        "(possibly as an alias); or implement the SchedulingConfigurer interface and call " +
                        "ScheduledTaskRegistrar#setScheduler explicitly within the configureTasks() callback: " +
                        ex2.getBeanNamesFound());
               }
            }
         }
         catch (NoSuchBeanDefinitionException ex2) {
            if (logger.isTraceEnabled()) {
               logger.trace("Could not find default ScheduledExecutorService bean - falling back to default: " +
                     ex2.getMessage());
            }
            // Giving up -> falling back to default scheduler within the registrar...
            logger.info("No TaskScheduler/ScheduledExecutorService bean found for scheduled processing");
         }
      }
   }

   this.registrar.afterPropertiesSet();
}
```

#### ScheduledTaskRegistrar

> 在上面的后置处理器中，最终调用了SchedulingConfigurer的实现类来修改ScheduledTaskRegistrar。

![img](http://www.xiewenshi.fans/blogimg/scheduledemo2.png)

这个类在属性填充完毕之后执行

@Override public void afterPropertiesSet() {   scheduleTasks(); }

#####  scheduleTasks

这个就是真正调度定时任务逻辑的地方

**//当我们没有配置配置类 设置taskschedule  那么这里默认new出来的就是单线程的线程池**

```Java
protected void scheduleTasks() {
//当我们没有配置配置类 设置taskschedule  那么这里默认newc出来的就是单线程的线程池
   if (this.taskScheduler == null) {
      this.localExecutor = Executors.newSingleThreadScheduledExecutor();
      this.taskScheduler = new ConcurrentTaskScheduler(this.localExecutor);
   }
   if (this.triggerTasks != null) {
      for (TriggerTask task : this.triggerTasks) {
         addScheduledTask(scheduleTriggerTask(task));
      }
   }
   if (this.cronTasks != null) {
      for (CronTask task : this.cronTasks) {
         addScheduledTask(scheduleCronTask(task));
      }
   }
   if (this.fixedRateTasks != null) {
      for (IntervalTask task : this.fixedRateTasks) {
         addScheduledTask(scheduleFixedRateTask(task));
      }
   }
   if (this.fixedDelayTasks != null) {
      for (IntervalTask task : this.fixedDelayTasks) {
         addScheduledTask(scheduleFixedDelayTask(task));
      }
   }
}
```

### 总结

- @EnableScheduling开启了ScheduledAnnotationBeanPostProcessor的配置
- ScheduledAnnotationBeanPostProcessor将任务交给了ScheduledTaskRegistrar
- SchedulingConfigurer可以实现对ScheduledTaskRegistrar的修改，如果没有实现类对其进行修改，那么默认的线程池就是单线程的。
- 配置自己的线程池为线程取一个独特的业务名字也便于排查问题。