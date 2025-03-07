+++ 
draft = false
date = "2025-03-07T00:00:00+03:00"
title = "Optimizing Moodle on ECS with Amazon RDS Proxy"

slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

Optimizing Moodle on ECS with Amazon RDS Proxy

While working on a Moodle Learning Management System (LMS) deployment on Amazon ECS, I noticed a critical issue — our database connections were being exhausted under high traffic, leading to unnecessary RDS scale-ups. Moodle wasn’t using connection pooling, which meant every request spawned a new connection, quickly depleting available resources.

To solve this, I integrated Amazon RDS Proxy in front of the database to optimize connection management and improve performance.

Implementing RDS Proxy in AWS CDK

To add RDS Proxy to our AWS Cloud Development Kit (CDK) stack, I modified the rds-stack.ts file with the following configuration:

{{< highlight ts >}}

this.proxy = new rds.DatabaseProxy(this, 'proxy', {
    proxyTarget: rds.ProxyTarget.fromInstance(this.db),
    secrets: [this.db.secret!],
    vpc: props.vpc,
    clientPasswordAuthType: rds.ClientPasswordAuthType.MYSQL_NATIVE_PASSWORD,
    maxConnectionsPercent: 90,
    maxIdleConnectionsPercent: 50,
    idleClientTimeout: cdk.Duration.minutes(30),
    requireTLS: true,
});

{{</ highlight >}}

After adding the proxy, I updated the moodle.ts construct to point to the RDS Proxy endpoint instead of the direct database endpoint.

{{< highlight ts >}}

export interface MoodleProps {
  vpc: ec2.IVpc

  systemAlertsTopic: sns.ITopic

  db: rds.IDatabaseInstance
  dbSecret: secretsmanager.ISecret
  proxy: rds.DatabaseProxy

  efs: efs.IFileSystem
  efsMoodleAccessPoint: efs.AccessPoint
  efsMoodleDataAccessPoint: efs.AccessPoint

  redis: elasticache.CfnReplicationGroup
  redisSg: ec2.ISecurityGroup

  ecsCluster: ecs.ICluster

  imageUri: string

  dbName: string
  dbUsername: string

  moodleUsername: string
  moodleEmail: string
  moodleSiteName: string

  domain: string
  hostedZoneId: string
  cfCert: acm.ICertificate
  wafAcl: wafv2.CfnWebACL
}

export class Moodle extends Construct {
  readonly taskDefinition: ecs.FargateTaskDefinition
  readonly service: ecs.FargateService

  constructor(scope: Construct, id: string, props: MoodleProps) {
    super(scope, id)

    const hostedZone = route53.HostedZone.fromHostedZoneAttributes(this, 'hosted-zone', {
      hostedZoneId: props.hostedZoneId,
      zoneName: props.domain,
    },
    )

    const cert = new acm.Certificate(this, 'cf-cert', {
      domainName: props.domain,
      certificateName: props.domain,
      validation: acm.CertificateValidation.fromDns(hostedZone),
    })


    const taskDefinition = new ecs.FargateTaskDefinition(this, 'task-def', {
      cpu: 4096,
      memoryLimitMiB: 8192,
    })

    const containerDefinition = taskDefinition.addContainer('container', {
      containerName: 'moodle',
      image: ecs.ContainerImage.fromRegistry(props.imageUri),
      portMappings: [{ containerPort: moodleContainerPort }],
      stopTimeout: cdk.Duration.seconds(120),
      environment: {
        'MOODLE_DATABASE_TYPE': 'mysqli', 
        'MOODLE_DATABASE_HOST': '<RDS Proxy Endpoint>',
        'MOODLE_DATABASE_PORT_NUMBER': '3306',
        'MOODLE_DATABASE_NAME': props.dbName,
        'MOODLE_DATABASE_USER': props.dbUsername,
        'MOODLE_USERNAME': props.moodleUsername,
        'MOODLE_EMAIL': props.moodleEmail,
        'MOODLE_SITE_NAME': props.moodleSiteName,
        'MOODLE_SKIP_BOOTSTRAP': 'no',
        'MOODLE_SKIP_INSTALL': 'no',
        'BITNAMI_DEBUG': 'true',
        'MOODLE_PASSWORD': "XXXXXXXXXXXXXX",
        'PHP_MAX_EXECUTION_TIME': '180',
        'PHP_POST_MAX_SIZE': '4192M',
        'PHP_UPLOAD_MAX_FILESIZE': '4192M',
      },
      secrets: {
        'MOODLE_DATABASE_PASSWORD': ecs.Secret.fromSecretsManager(props.dbSecret!, 'password'),
      },
      logging: ecs.LogDrivers.awsLogs({ streamPrefix: 'ecs', logRetention: 7 })
    })

{{</ highlight >}}

Updating Moodle Configuration

To test the setup, I connected to the debug container, which we use for troubleshooting when Moodle crashes, and manually updated Moodle’s database configuration.

I accessed the debug container with:

{{< highlight bash >}}

AWS_PROFILE=neu-sso aws ecs execute-command  \
    --region eu-west-3 \
    --cluster moodle-ecs-cluster \
    --task <task-id> \
    --container debug \
    --command "/bin/sh" \
    --interactive

{{</ highlight >}}

Inside the container, I updated /bitnami/moodle/config.php:

{{< highlight bash >}}

$CFG->dbhost = '<RDS Proxy Endpoint>';

{{</ highlight >}}

Resolving SSL Issues

Initially, I had requireTLS set to true, but this caused issues since Moodle requires special configuration to handle TLS for MySQL connections. Rather than modifying Moodle's internals, I disabled TLS in RDS Proxy, ensuring that both the application and the database were in a secure VPC:


{{< highlight bash >}}

this.proxy = new rds.DatabaseProxy(this, 'proxy', {
    proxyTarget: rds.ProxyTarget.fromInstance(this.db),
    secrets: [this.db.secret!],
    vpc: props.vpc,
    clientPasswordAuthType: rds.ClientPasswordAuthType.MYSQL_NATIVE_PASSWORD,
    maxConnectionsPercent: 90,
    maxIdleConnectionsPercent: 50,
    idleClientTimeout: cdk.Duration.minutes(30),
    requireTLS: false,
});

{{</ highlight >}}

After making these changes, I redeployed the ECS service, forcing it to pick up the new database configuration.

Conclusion

By integrating RDS Proxy, I significantly improved database connection handling for Moodle on ECS, preventing unnecessary RDS scale-ups due to connection exhaustion.

With this setup, Moodle can now efficiently handle high traffic while maintaining stable database connections.

🙋🏼‍♀️ Additionally, another way that enabling database persistence also helped resolve some connection issues by [ensuring persistent database connections](https://github.com/moodle/moodle/blob/5670447ece698da18e20d0f1984965535487feda/config-dist.php#L49) within Moodle’s configuration.