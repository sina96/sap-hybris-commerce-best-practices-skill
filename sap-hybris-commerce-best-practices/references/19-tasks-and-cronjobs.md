# Writing and Executing Tasks

SAP Commerce Cloud provides two main mechanisms for executing background tasks: **CronJobs** (comprehensive scheduled jobs) and **Task Service** (lightweight task scheduling). This guide covers both approaches.

## Overview

### CronJobs vs Tasks

| Feature | CronJobs | Task Service |
|---------|----------|--------------|
| Complexity | Full-featured with history, triggers, scheduling | Lightweight and simple |
| Use Case | Complex, long-running background processes | Simple, quick actions |
| History | Detailed execution logs and history | Minimal logging |
| Scheduling | Cron expressions, flexible triggers | Simple time-based or event-based |
| Abortable | Yes (with implementation) | Limited |
| Cluster Support | Node groups, distributed execution | Basic cluster support |

### When to Use Each

**Use CronJobs when:**
- You need detailed execution history
- Complex scheduling (cron expressions)
- Long-running processes
- Need to monitor and abort jobs
- Running on specific node groups

**Use Task Service when:**
- Simple, quick actions
- Event-driven execution
- Minimal overhead required
- Don't need execution history

---

## CronJobs

### Architecture

```
Job (logic) + Trigger (schedule) = CronJob (execution instance)
```

**Components:**
1. **JobPerformable**: Interface defining the task logic
2. **Job**: Type definition in items.xml
3. **Trigger**: Scheduling configuration
4. **CronJob**: Execution instance with history

### Creating a CronJob

#### 1. Implement JobPerformable

```java
public class ProductSyncJob implements JobPerformable<CronJobModel> {
    
    @Autowired
    private ProductService productService;
    
    @Autowired
    private SessionService sessionService;
    
    @Override
    public PerformResult perform(CronJobModel cronJob) {
        try {
            // Get parameters from cronJob
            String catalogCode = cronJob.getJob().getCode();
            
            // Execute business logic
            productService.synchronizeProducts(catalogCode);
            
            return new PerformResult(CronJobResult.SUCCESS, CronJobStatus.FINISHED);
        } catch (Exception e) {
            return new PerformResult(CronJobResult.ERROR, CronJobStatus.ABORTED);
        }
    }
    
    @Override
    public boolean isAbortable() {
        return true; // Allow job to be aborted
    }
}
```

#### 2. Define Types in items.xml

```xml
<itemtype code="ProductSyncJob" autocreate="true" generate="true"
          extends="ServicelayerJob"
          jaloclass="com.mycompany.core.jalo.ProductSyncJob">
</itemtype>

<itemtype code="ProductSyncCronJob" autocreate="true" generate="true"
          extends="CronJob">
    <attributes>
        <attribute qualifier="catalogCode" type="java.lang.String">
            <persistence type="property"/>
        </attribute>
    </attributes>
</itemtype>
```

#### 3. Spring Configuration

```xml
<bean id="productSyncJob" class="com.mycompany.core.job.ProductSyncJob"/>

<bean id="productSyncJobPerformable" 
      class="de.hybris.platform.servicelayer.cronjob.impl.DefaultJobPerformableAdapter">
    <property name="jobPerformable" ref="productSyncJob"/>
</bean>
```

#### 4. Create and Schedule via ImpEx

```impex
# Create Job
INSERT_UPDATE ServicelayerJob;code[unique=true];springId
;productSyncJob;productSyncJob

# Create Trigger (runs daily at 2 AM)
INSERT_UPDATE Trigger;cronJob(code)[unique=true];cronExpression;active
;productSyncCronJob;0 0 2 * * ?;true

# Create CronJob
INSERT_UPDATE ProductSyncCronJob;code[unique=true];job(code);catalogCode;sessionLanguage(isocode)
;productSyncCronJob;productSyncJob;electronics;en
```

### Cron Expression Format
 
```
+---------------- second (0-59)
| +-------------- minute (0-59)
| | +------------ hour (0-23)
| | | +---------- day of month (1-31)
| | | | +-------- month (1-12)
| | | | | +------ day of week (0-6, SUN-SAT)
| | | | | |
* * * * * *
```

**Common Patterns:**
- `0 0 2 * * ?` - Daily at 2:00 AM
- `0 0/30 * * * ?` - Every 30 minutes
- `0 0 0 * * SUN` - Every Sunday at midnight
- `0 0 0 1 * ?` - First day of every month

### Abortable Jobs

To make a job abortable, implement checks at strategic points:

```java
public class LongRunningJob implements JobPerformable<CronJobModel> {
    
    @Autowired
    private CronJobService cronJobService;
    
    @Override
    public PerformResult perform(CronJobModel cronJob) {
        List<ItemModel> items = getItemsToProcess();
        
        for (ItemModel item : items) {
            // Check if job should abort
            if (cronJobService.isAborting(cronJob)) {
                return new PerformResult(CronJobResult.UNKNOWN, CronJobStatus.ABORTED);
            }
            
            processItem(item);
            
            // Update progress
            cronJob.setProgress((processedCount / totalCount) * 100);
            modelService.save(cronJob);
        }
        
        return new PerformResult(CronJobResult.SUCCESS, CronJobStatus.FINISHED);
    }
    
    @Override
    public boolean isAbortable() {
        return true;
    }
}
```

### Composite CronJobs

Run multiple jobs sequentially:

```impex
INSERT_UPDATE CompositeCronJob;code[unique=true];jobs(code)
;dailyMaintenanceJob;cleanupCronJob,solrIndexCronJob,backupCronJob
```

### Cluster Configuration

Run on specific node groups:

```impex
INSERT_UPDATE Trigger;cronJob(code)[unique=true];cronExpression;active;nodeGroup
;productSyncCronJob;0 0 2 * * ?;true;backgroundProcessing
```

---

## Task Service

### Overview

The Task Service provides a lightweight alternative to CronJobs for simple scheduling needs.

### Creating a Task

#### 1. Implement TaskRunner

```java
@Service("productUpdateTaskRunner")
public class ProductUpdateTaskRunner implements TaskRunner<TaskModel> {
    
    @Autowired
    private ProductService productService;
    
    @Override
    public void run(TaskService taskService, TaskModel task) throws RetryLaterException {
        String productCode = task.getContext();
        
        try {
            ProductModel product = productService.getProductForCode(productCode);
            // Update product
            productService.save(product);
        } catch (Exception e) {
            // Retry after 5 minutes
            throw new RetryLaterException(e, 5, TimeUnit.MINUTES);
        }
    }
    
    @Override
    public void handleError(TaskService taskService, TaskModel task, Throwable error) {
        // Log error, send notification, etc.
        LOG.error("Task execution failed: " + task.getPk(), error);
    }
}
```

#### 2. Create and Execute Tasks

```java
@Autowired
private TaskService taskService;

@Autowired
private ModelService modelService;

public void scheduleProductUpdate(String productCode) {
    TaskModel task = modelService.create(TaskModel.class);
    task.setRunnerBean("productUpdateTaskRunner");
    task.setContext(productCode);
    task.setExecutionDate(new Date(System.currentTimeMillis() + 60000)); // 1 minute from now
    
    taskService.scheduleTask(task);
}
```

### Event-Based Tasks

Execute tasks in response to events:

```java
@EventListener
public void onOrderPlaced(OrderPlacedEvent event) {
    TaskModel task = modelService.create(TaskModel.class);
    task.setRunnerBean("orderProcessingTaskRunner");
    task.setContext(event.getOrder().getCode());
    task.setExecutionDate(new Date());
    
    taskService.scheduleTask(task);
}
```

### Running on Specific Nodes

Force execution on backgroundProcessing node:

```java
private void startOnBackgroundProcessingNode(String taskName) {
    if (!clusterService.isClusteringEnabled()) {
        // Run locally
        executeTask(taskName);
        return;
    }
    
    TaskModel task = modelService.create(TaskModel.class);
    task.setRunnerBean("myTaskRunner");
    task.setExecutionDate(new Date());
    task.setContext(taskName);
    task.setNodeGroup("backgroundProcessing");
    
    taskService.scheduleTask(task);
}
```

---

## Best Practices

### DO

**CronJobs:**
- Make long-running jobs abortable
- Use node groups for resource-intensive jobs
- Set appropriate session context (language, currency)
- Handle exceptions gracefully with proper error results
- Clean up old CronJob history regularly
- Use composite jobs for related operations
- Test jobs thoroughly before scheduling in production

**Tasks:**
- Use for simple, quick operations
- Implement proper error handling with RetryLaterException
- Keep task logic lightweight
- Use context field to pass parameters
- Consider using for event-driven processing

### DON'T

- Don't perform heavy operations in Task Service (use CronJobs)
- Don't ignore error handling - always return appropriate PerformResult
- Don't run resource-intensive jobs on storefront nodes
- Don't create jobs without proper testing
- Don't forget to configure session context for localized operations
- Don't accumulate unlimited CronJob history

### Data Cleanup

Regularly clean up old CronJob data:

```impex
$twoWeeks=1209600

INSERT_UPDATE FlexibleSearchRetentionRule;code[unique=true];searchQuery;retentionTimeSeconds;actionReference
;cronjobCleanupRule;"select {c:pk}, {c:itemType} from {CronJob as c join ComposedType as t on {c:itemtype} = {t:pk} left join Trigger as trg on {trg:cronjob} = {c:pk}} where {trg:pk} is null and {c:code} like '00______%' and {c:creationTime} < ?CALENDAR.TIME[addSeconds(-$twoWeeks)]";1209600;basicRemoveCleanupAction

INSERT_UPDATE RetentionJob;code[unique=true];retentionRules(code)
;cronjobRetentionJob;cronjobCleanupRule

INSERT_UPDATE CronJob;code[unique=true];job(code);sessionLanguage(isocode)
;cronjobCleanupCronJob;cronjobRetentionJob;en

INSERT_UPDATE Trigger;cronJob(code)[unique=true];cronExpression;active
;cronjobCleanupCronJob;0 0 3 * * ?;true
```

### Testing

**Unit Test:**
```java
@UnitTest
public class ProductSyncJobTest {
    
    @Mock
    private ProductService productService;
    
    @InjectMocks
    private ProductSyncJob job;
    
    @Test
    public void testPerformSuccess() {
        CronJobModel cronJob = mock(CronJobModel.class);
        ServicelayerJob jobModel = mock(ServicelayerJob.class);
        when(cronJob.getJob()).thenReturn(jobModel);
        when(jobModel.getCode()).thenReturn("electronics");
        
        PerformResult result = job.perform(cronJob);
        
        assertEquals(CronJobResult.SUCCESS, result.getResult());
        assertEquals(CronJobStatus.FINISHED, result.getStatus());
    }
}
```

**Integration Test:**
```java
@IntegrationTest
public class ProductSyncJobIntegrationTest extends ServicelayerTest {
    
    @Resource
    private CronJobService cronJobService;
    
    @Resource
    private ModelService modelService;
    
    @Test
    public void testCronJobExecution() {
        // Create and configure CronJob
        CronJobModel cronJob = createTestCronJob();
        
        // Execute
        cronJobService.performCronJob(cronJob, true);
        
        // Verify results
        assertEquals(CronJobResult.SUCCESS, cronJob.getResult());
    }
}
```

---

## Quick Commands

```bash
# Execute CronJob via HAC
# Platform -> CronJobs -> Find your job -> Start

# Execute via ImpEx
"#%impex.setCronJob(\"myCronJob\", true)"

# List running CronJobs via Groovy
def runningJobs = cronJobService.getRunningCronJobs()
runningJobs.each { println it.code }

# Abort a CronJob via Groovy
def cronjob = defaultCronJobService.getCronJob('myCronJob')
if (defaultCronJobService.isAbortable(cronjob)) {
    defaultCronJobService.abortCronJob(cronjob)
}
```

## Resources

- **HAC**: http://localhost:9001/hac -> Platform -> CronJobs
- **Backoffice**: System -> CronJobs
- **SAP Help**: help.sap.com -> CronJobs and Task Service documentation
