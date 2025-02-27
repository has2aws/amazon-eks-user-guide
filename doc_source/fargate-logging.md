# Fargate logging<a name="fargate-logging"></a>

Amazon EKS on Fargate offers a built\-in log router based on Fluent Bit\. This means that you don't explicitly run a Fluent Bit container as a sidecar, but Amazon runs it for you\. All that you have to do is configure the log router\. The configuration happens through a dedicated `ConfigMap` that must meet the following criteria:
+ Named `aws-logging`
+ Created in a dedicated namespace called `aws-observability`

Once you've created the `ConfigMap`, Amazon EKS on Fargate automatically detects it and configures the log router with it\. Fargate uses a version of AWS for Fluent Bit, an upstream compliant distribution of Fluent Bit managed by AWS\. For more information, see [AWS for Fluent Bit](https://github.com/aws/aws-for-fluent-bit) on GitHub\.

The log router allows you to use the breadth of services at AWS for log analytics and storage\. You can stream logs from Fargate directly to Amazon CloudWatch, Amazon OpenSearch Service\. You can also stream logs to destinations such as Amazon S3, Amazon Kinesis Data Streams, and partner tools through Amazon Kinesis Data Firehose\.

**Prerequisites**
+ An existing Fargate profile that specifies an existing Kubernetes namespace that you deploy Fargate pods to\. For more information, see [Create a Fargate profile for your cluster](fargate-getting-started.md#fargate-gs-create-profile)\.
+ An existing Fargate pod execution role\. For more information, see [Create a Fargate pod execution role](fargate-getting-started.md#fargate-sg-pod-execution-role)\.

## Log router configuration<a name="fargate-logging-log-router-configuration"></a>

**To configure the log router**

In the following steps, replace the `<example values>` \(including `<>`\) with your own values\.

1. Create a dedicated Kubernetes namespace named `aws-observability`\.

   1. Save the following contents to a file named `aws-observability-namespace.yaml` on your computer\. The value for `name` must be `aws-observability` and the `aws-observability: enabled` label is required\.

      ```
      kind: Namespace
      apiVersion: v1
      metadata:
        name: aws-observability
        labels:
          aws-observability: enabled
      ```

   1. Create the namespace\.

      ```
      kubectl apply -f aws-observability-namespace.yaml
      ```

1. Create a `ConfigMap` with a `Fluent Conf` data value to ship container logs to a destination\. Fluent Conf is Fluent Bit, which is a fast and lightweight log processor configuration language that is used to route container logs to a log destination of your choice\. For more information, see [Configuration File](https://docs.fluentbit.io/manual/administration/configuring-fluent-bit/configuration-file) in the Fluent Bit documentation\. 
**Important**  
In a typical `Fluent Conf` the main sections included are `Service`, `Input`, `Filter`, and `Output`\. The Fargate log router however, only accepts:  
 The `Filter` and `Output` sections and manages the `Service` and `Input` sections itself\. 
A `Parser` section\.
If you provide any sections other than `Filter`, `Output`, and `Parser`, the sections are rejected\.

   When creating the `ConfigMap`, take into account the following rules that Fargate uses to validate fields:
   + `[FILTER]`, `[OUTPUT]`, and `[PARSER]` are supposed to be specified under each corresponding key\. For example, `[FILTER]` must be under `filters.conf`\. You can have one or more `[FILTER]`s under filters\.conf\. The `[OUTPUT]` and `[PARSER]` sections should also be under their corresponding keys\. By specifying multiple `[OUTPUT]` sections, you can route your logs to different destinations at the same time\.
   + Fargate validates the required keys for each section\. `Name` and `match` are required for each `[FILTER]` and `[OUTPUT]`\. `Name` and `format` are required for each `[PARSER]`\. The keys are case\-insensitive\.
   + Environment variables such as `${ENV_VAR}` are not allowed in the `ConfigMap`\.
   + The indentation has to be the same for either directive or key\-value pair within each `filters.conf`, `output.conf`, and `parsers.conf`\. Key\-value pairs have to be indented more than directives\.
   + Fargate validates against the following supported filters: `grep`, `parser`, `record_modifier`, `rewrite_tag`, `throttle`, `nest`, and `modify`\.
   + Fargate validates against the following supported output: `es`, `firehose`, `kinesis_firehose`, `cloudwatch`, `cloudwatch_logs`, and `kinesis`\. 
   + At least one supported `Output` plugin has to be provided in the `ConfigMap` to enable logging\. `Filter` and `Parser` are not required to enable logging\.

   You can also run Fluent Bit on Amazon EC2 using the desired configuration to troubleshoot any issues that arise from validation\. Create your `ConfigMap` using one of the following examples\.
**Important**  
Amazon EKS Fargate logging doesn't support dynamic configuration of `ConfigMaps`\. Any changes to `ConfigMaps` are applied to new pods only\. Changes aren't applied to existing pods\.

   Create a `ConfigMap` using the example for your desired log destination\.

------
#### [ CloudWatch ]

   You have two output options when using CloudWatch:
   + [An output plugin written in C](https://docs.fluentbit.io/manual/v/1.5/pipeline/outputs/cloudwatch) 
   + [An output plugin written in Golang](https://github.com/aws/amazon-cloudwatch-logs-for-fluent-bit)

   The following example shows you how to use the `cloudwatch_logs` plugin to send logs to CloudWatch\.

   1. Save the following contents to a file named `aws-logging-cloudwatch-configmap.yaml`\. The parameters under `[OUTPUT]` are required\.

      ```
      kind: ConfigMap
      apiVersion: v1
      metadata:
        name: aws-logging
        namespace: aws-observability
      data:
        output.conf: |
          [OUTPUT]
              Name cloudwatch_logs
              Match   *
              region us-east-1
              log_group_name fluent-bit-cloudwatch
              log_stream_prefix from-fluent-bit-
              auto_create_group true
      
        parsers.conf: |
          [PARSER]
              Name crio
              Format Regex
              Regex ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>P|F) (?<log>.*)$
              Time_Key    time
              Time_Format %Y-%m-%dT%H:%M:%S.%L%z
        
        filters.conf: |
           [FILTER]
              Name parser
              Match *
              Key_name log
              Parser crio
              Reserve_Data On
              Preserve_Key On
      ```

   1. Apply the manifest to your cluster\.

      ```
      kubectl apply -f aws-logging-cloudwatch-configmap.yaml
      ```

   1. Download the CloudWatch IAM policy to your computer\. You can also [view the policy](https://raw.githubusercontent.com/aws-samples/amazon-eks-fluent-logging-examples/mainline/examples/fargate/cloudwatchlogs/permissions.json) on GitHub\.

      ```
      curl -o permissions.json https://raw.githubusercontent.com/aws-samples/amazon-eks-fluent-logging-examples/mainline/examples/fargate/cloudwatchlogs/permissions.json
      ```

------
#### [ Amazon OpenSearch Service ]

   If you want to send logs to Amazon OpenSearch Service\. You can use [es](https://docs.fluentbit.io/manual/v/1.5/pipeline/outputs/elasticsearch) output, which is a plugin written in C\. The following example shows you how to use the plugin to send logs to OpenSearch\.

   1. Save the following contents to a file named `aws-logging-opensearch-configmap.yaml`\.

      ```
      kind: ConfigMap
      apiVersion: v1
      metadata:
        name: aws-logging
        namespace: aws-observability
      data:
        output.conf: |
          [OUTPUT]
            Name  es
            Match *
            Host  search-example-gjxdcilagiprbglqn42jsty66y.us-east-1.es.amazonaws.com
            Port  443
            Index example
            Type  example_type
            AWS_Auth On
            AWS_Region us-east-1
            tls   On
      ```

   1. Apply the manifest to your cluster\.

      ```
      kubectl apply -f aws-logging-opensearch-configmap.yaml
      ```

   1. Download the OpenSearch IAM policy to your computer\. You can also [view the policy](https://raw.githubusercontent.com/aws-samples/amazon-eks-fluent-logging-examples/mainline/examples/fargate/amazon-elasticsearch/permissions.json) on GitHub\.

      ```
      curl -o permissions.json https://raw.githubusercontent.com/aws-samples/amazon-eks-fluent-logging-examples/mainline/examples/fargate/amazon-elasticsearch/permissions.json
      ```

      Make sure that OpenSearch Dashboards' access control is configured properly\. The `all_access role` in OpenSearch Dashboards needs to have the Fargate pod execution role as well as the IAM role mapped\. The same mapping must be done for the `security_manager` role\. You can add the previous mappings by selecting `Menu`, then `Security`, then `Roles`, and then select the respective roles\. For more information, see [How do I troubleshoot CloudWatch Logs so that it streams to my Amazon ES domain?](http://aws.amazon.com/tr/premiumsupport/knowledge-center/es-troubleshoot-cloudwatch-logs/)\.

------
#### [ Kinesis Data Firehose ]

   Send logs to Kinesis Data Firehose\. You have two output options when using Kinesis Data Firehose:
   + [https://docs.fluentbit.io/manual/pipeline/outputs/firehose](https://docs.fluentbit.io/manual/pipeline/outputs/firehose) – An output plugin written in C\.
   + [https://github.com/aws/amazon-kinesis-firehose-for-fluent-bit](https://github.com/aws/amazon-kinesis-firehose-for-fluent-bit) – An output plugin written in Golang\.

   The following example shows you how to use the `kinesis_firehose` plugin to send logs to Kinesis Data Firehose\.

   1. Save the following contents to a file named `aws-logging-firehose-configmap.yaml`\.

      ```
      kind: ConfigMap
      apiVersion: v1
      metadata:
        name: aws-logging
        namespace: aws-observability
      data:
        output.conf: |
          [OUTPUT]
           Name  kinesis_firehose
           Match *
           region us-east-1
           delivery_stream my-stream-firehose
      ```

   1. Apply the manifest to your cluster\.

      ```
      kubectl apply -f aws-logging-firehose-configmap.yaml
      ```

   1. Download the Kinesis Data Firehose IAM policy to your computer\. You can also [view the policy](https://raw.githubusercontent.com/aws-samples/amazon-eks-fluent-logging-examples/mainline/examples/fargate/kinesis-firehose/permissions.json) on GitHub\.

      ```
      curl -o permissions.json https://raw.githubusercontent.com/aws-samples/amazon-eks-fluent-logging-examples/mainline/examples/fargate/kinesis-firehose/permissions.json
      ```

------

1. Create an IAM policy from the policy file you downloaded in a previous step\.

   ```
   aws iam create-policy --policy-name eks-fargate-logging-policy --policy-document file://permissions.json
   ```

1. Attach the IAM policy to the pod execution role specified for your Fargate profile\. Replace `111122223333` with your account ID\.

   ```
   aws iam attach-role-policy \
     --policy-arn arn:aws:iam::111122223333:policy/eks-fargate-logging-policy \
     --role-name your-pod-execution-role
   ```

## Test application<a name="fargate-logging-test-application"></a>

1. Deploy a sample pod\.

   1. Save the following contents to a file named *sample\-app*\.yaml on your computer\.

      ```
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: sample-app
        namespace: same-namespace-as-your-fargate-profile
      spec:
        replicas: 3
        selector:
          matchLabels:
            app: nginx
        template:
          metadata:
            labels:
              app: nginx
          spec:
            containers:
              - name: nginx
                image: nginx:latest
                ports:
                  - name: http
                    containerPort: 80
      ```

   1. Apply the manifest to the cluster\.

      ```
      kubectl apply -f sample-app.yaml
      ```

1. View the NGINX logs using the destination\(s\) that you configured in the `ConfigMap`\.

## Size considerations<a name="fargate-logging-size-considerations"></a>

We suggest that you plan for up to 50 MB of memory for the log router\. If you expect your application to generate logs at very high throughput then you should plan for up to 100 MB\.

## Troubleshooting<a name="fargate-logging-troubleshooting"></a>

To confirm whether the logging feature is enabled or disabled for some reason, such as an invalid `ConfigMap`, and why it is invalid, check your pod events with `kubectl describe pod pod_name`\. The output might include pod events that clarify whether logging is enabled or not, such as the following example output\.

```
...
Annotations:          CapacityProvisioned: 0.25vCPU 0.5GB
                      Logging: LoggingDisabled: LOGGING_CONFIGMAP_NOT_FOUND
                      kubernetes.io/psp: eks.privileged
...
Events:
  Type     Reason           Age        From                                                           Message
  ----     ------           ----       ----                                                           -------
  Warning  LoggingDisabled  <unknown>  fargate-scheduler                                              Disabled logging because aws-logging configmap was not found. configmap "aws-logging" not found
```

The pod events are ephemeral with a time period depending on the settings\. You can also view a pod's annotations using `kubectl describe pod pod-name`\. In the pod annotation, there is information about whether the logging feature is enabled or disabled and the reason\.