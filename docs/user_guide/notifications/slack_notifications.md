# Slack Notifications

Spark Expectations supports sending notifications to Slack channels via webhooks when data quality checks are performed. This enables teams to stay informed about data quality issues in real-time.

By default, Slack notifications are disabled. To enable them, you need to configure the required parameters and set up a Slack webhook URL.

## Prerequisites

Before configuring Slack notifications in Spark Expectations, you need to:

1. **Create a Slack App** in your workspace
2. **Enable Incoming Webhooks** for your Slack app
3. **Create a webhook URL** for the target channel
4. **Copy the webhook URL** to use in your configuration

### Setting up Slack Webhook

1. Go to [Slack API](https://api.slack.com/apps) and create a new app
2. Navigate to **Incoming Webhooks** in your app settings
3. Turn on **Activate Incoming Webhooks**
4. Click **Add New Webhook to Workspace**
5. Select the channel where you want to receive notifications
6. Copy the generated webhook URL (it looks like: `https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX`)

## Configuration Parameters

### Required Parameters

!!! info "user_config.se_notifications_enable_slack"
    Master toggle to enable Slack notifications. Set to `True` to activate Slack notifications.

!!! info "user_config.se_notifications_slack_webhook_url" 
    The Slack webhook URL obtained from your Slack app configuration. This is where notifications will be sent.

### Notification Triggers

These parameters control **when** Slack notifications are sent during Spark-Expectations runs:

- <abbr title="Enable notifications when job starts">user_config.se_notifications_on_start</abbr>
- <abbr title="Enable notifications when job ends">user_config.se_notifications_on_completion</abbr> 
- <abbr title="Enable notifications on failure">user_config.se_notifications_on_fail</abbr>
- <abbr title="Notify if error drop threshold is breached">user_config.se_notifications_on_error_drop_exceeds_threshold_breach</abbr>
- <abbr title="Notify if rules with action 'ignore' fail">user_config.se_notifications_on_rules_action_if_failed_set_ignore</abbr>
- <abbr title="Threshold value for error drop notifications">user_config.se_notifications_on_error_drop_threshold</abbr>

## Configuration Example

Here's how to configure Slack notifications in your Spark Expectations setup:

```python
from spark_expectations.config.user_config import *

# Basic Slack configuration
user_configs = {
    # Enable Slack notifications
    se_notifications_enable_slack: True,
    
    # Slack webhook URL (replace with your actual webhook URL)
    se_notifications_slack_webhook_url: "https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX",
    
    # Configure when to send notifications
    se_notifications_on_start: True,
    se_notifications_on_completion: True, 
    se_notifications_on_fail: True,
    se_notifications_on_error_drop_exceeds_threshold_breach: True,
    se_notifications_on_error_drop_threshold: 15,
    
    # Other required configurations
    se_enable_error_table: True,
    se_dq_rules_params: {
        "env": "dev",
        "table": "your_table_name",
        "data_object_name": "your_data_object",
        "data_source": "your_data_source",
    }
}
```

## Complete Example

Here's a complete example showing how to integrate Slack notifications with your data quality pipeline:

```python
from pyspark.sql import SparkSession
from spark_expectations.core.expectations import SparkExpectations, WrappedDataFrameWriter
from spark_expectations.config.user_config import *

# Initialize Spark session
spark = SparkSession.builder.appName("DataQualityWithSlack").getOrCreate()

# Configure Spark Expectations with Slack notifications
user_configs = {
    # Slack notification settings
    se_notifications_enable_slack: True,
    se_notifications_slack_webhook_url: "https://hooks.slack.com/services/YOUR/WEBHOOK/URL",
    
    # Notification triggers
    se_notifications_on_start: True,
    se_notifications_on_completion: True,
    se_notifications_on_fail: True,
    se_notifications_on_error_drop_exceeds_threshold_breach: True,
    se_notifications_on_error_drop_threshold: 10,
    
    # Data quality settings
    se_enable_error_table: True,
    se_enable_query_dq_detailed_result: True,
    se_enable_agg_dq_detailed_result: True,
    
    # Environment configuration
    se_dq_rules_params: {
        "env": "production",
        "table": "customer_orders", 
        "data_object_name": "orders_pipeline",
        "data_source": "order_system",
    }
}

# Initialize Spark Expectations
se = SparkExpectations(
    product_id="your_product",
    rules_df=spark.table("rules_table"),
    stats_table="dq_spark.statistics_table",
    stats_table_writer=WrappedDataFrameWriter().mode("append").format("delta"),
    target_and_error_table_writer=WrappedDataFrameWriter().mode("append").format("delta"),
    debugger=False,
    **user_configs
)

# Your data processing with quality checks
input_df = spark.table("source_table")

# Apply data quality expectations
output_df = se.with_expectations(
    dataset=input_df,
    expectation_suite="your_expectation_suite",
    write_to_table="target_table",
    write_to_temp_table="temp_table"
)
```

## Message Format

Slack notifications sent by Spark Expectations include:

- **Job Status**: Whether the data quality check started, completed, or failed
- **Data Quality Results**: Summary of passed/failed expectations  
- **Error Details**: Information about specific data quality issues
- **Metadata**: Table name, environment, timestamp, and other contextual information

## Troubleshooting

### Common Issues

1. **Webhook URL Invalid**
   - Verify your webhook URL is correct and active
   - Test the webhook URL directly using curl or Postman

2. **Notifications Not Received**
   - Check if `se_notifications_enable_slack` is set to `True`
   - Ensure the appropriate trigger flags are enabled
   - Verify network connectivity from your Spark environment

3. **Permission Errors** 
   - Ensure your Slack app has permission to post to the target channel
   - Verify the webhook hasn't been revoked or expired

### Testing Slack Integration

You can test your Slack webhook configuration using curl:

```bash
curl -X POST -H 'Content-type: application/json' \
--data '{"text":"Test message from Spark Expectations"}' \
YOUR_WEBHOOK_URL
```

## Security Considerations

!!! warning "Webhook URL Security"
    - Store webhook URLs securely (use environment variables or secret management)
    - Rotate webhook URLs periodically
    - Limit webhook permissions to specific channels
    - Monitor webhook usage in Slack app logs

## Best Practices

1. **Channel Strategy**: Use dedicated channels for data quality notifications
2. **Message Frequency**: Configure appropriate trigger thresholds to avoid notification spam
3. **Environment Separation**: Use different webhooks for different environments (dev, staging, prod)
4. **Error Handling**: Monitor notification failures and implement fallback mechanisms
5. **Testing**: Test notifications in development before deploying to production

## Advanced Configuration

For more complex notification scenarios, you can:

- Combine Slack with email notifications for redundancy
- Use different webhooks for different types of notifications
- Implement custom message formatting through templates
- Set up notification routing based on data quality severity levels